---
title: "메모리는 절반으로, 속도는 1.5배 빠르게 한 이야기 - hayakoe Part 2"
slug: 137243ede98030f1
date: 2026-04-26T17:40:46+09:00
image: "cover.png"
categories:
    - AI
    - TTS
tags:
    - AI
    - TTS
    - ONNX
    - 양자화
    - Style-Bert-VITS2
    - Python
    - 최적화
---

hayakoe 시리즈의 두 번째 본편입니다. 이전 글은 [hayakoe - 2년만에 다시 TTS 모델 이것저것 보다 또 다시 VITS로 정착한 이야기 - Part 1](/p/4c2a6225a737f531/) 에서 보실 수 있습니다.

[Part 1](/p/4c2a6225a737f531/) 에서 학습된 모델까지 손에 쥐었지만, 시리즈 [서론](/p/ea78fbb9b78c38ac/) 에서 짚었듯 그 모델을 그대로 쓰기엔 여러 가지 불편이 있었습니다 — `openjtalk` 가 외부에서 사전 데이터를 받아와 그 서버가 죽으면 같이 죽는 SPOF 문제, 그 의존성 때문에 막혀 있던 Python 버전업, 영어 합성이 안 돼 번역 레이어로 우회 중이던 점, 그리고 본 글에서 다룰 메모리·속도 부담까지.

Part 2 에서는 이 중에서 **메모리와 속도** 를 어떻게 줄였는지를 풀어 봅니다. 구체적으로는:

- **메모리**: 1 화자 로드 시 RAM **5,122 MB**
- **속도**: 38.5 초 분량 텍스트 합성에 35.3 초 — 배속 **1.09×** (CPU FP32 기준)

알람·브리핑 용도로 쓰려면 둘 다 부담스러운 수치였습니다. Part 2 는 이 두 숫자를 어떻게 **2,346 MB · 3.6×** 까지 끌어내렸는지에 대한 이야기입니다.

큰 흐름은 다음과 같습니다.

1. **병목 측정** — 시간을 어디서 쓰는지부터
2. **BERT 양자화** — 속도가 아니라 메모리에 효과
3. **Synthesizer 는 양자화 불가** — Flow 레이어가 깨짐
4. **ONNX Runtime 그래프 최적화** — Synthesizer 속도 회수
5. **메모리 89 MB 사건** — 측정의 함정

### 1. 어디가 병목인지부터

처음 떠올렸던 가설은 단순했습니다. **weight 파일 크기를 보면 BERT (DeBERTa v2 Large JP) 가 모델 전체의 약 84% 를 차지**하니, BERT 만 양자화해도 메모리·속도 모두 잡힐 것이라 기대했습니다.

그런데 그 가설을 검증해 보려면, 시간이 정확히 어디서 소비되는지부터 측정해야 했습니다. BERT 와 Synthesizer (VITS) 의 추론 시간을 분리해 측정해 봤습니다 (`time.perf_counter` 5 회 평균, PyTorch FP32 / CPU 기준).

| 텍스트 | BERT | Synthesizer | BERT 비중 | Synth 비중 |
|---|---|---|---|---|
| short (1.7s) | 0.489 s | 0.885 s | 36 % | **64 %** |
| medium (5.3s) | 0.602 s | 2.504 s | 19 % | **81 %** |
| long (7.8s) | 0.690 s | 3.714 s | 16 % | **84 %** |
| xlong (30s) | 1.074 s | 11.410 s | 9 % | **91 %** |

기대와는 완전히 반대 결과였습니다. **CPU 시간의 64 ~ 91 % 를 Synthesizer 가 가져가고**, 텍스트가 길어질수록 그 비중이 더 커졌습니다.

이유는 단순합니다. BERT 는 입력 텍스트 길이에 비교적 둔감한 반면, Synthesizer 는 생성할 오디오 길이에 비례해 시간이 늘어나기 때문입니다. 텍스트가 길수록 합성해야 할 오디오 frame 수가 많아지고, Synthesizer 의 Conv1d 레이어들이 그 frame 만큼 반복 호출됩니다.

즉, 케이스를 나눠서 보면:

- **짧은 텍스트** — BERT 가 36 % 정도 비중을 차지하긴 하지만, 어차피 추론 전체가 1 초 남짓이라 BERT 를 양자화해도 체감 차이가 거의 없습니다.
- **긴 텍스트** — Synthesizer 가 91 % 까지 가져갑니다. 여기서 BERT 만 빠르게 만들어 봤자 줄일 수 있는 게 9 % 뿐이라, 실질적인 가속에는 큰 의미가 없습니다.

어느 쪽이든 BERT 만 잡아서는 **체감 가능한 속도 향상**을 만들기 어려웠습니다.

> "측정하지 않은 최적화는 직감에 의존하고, 직감은 자주 틀린다" 는 격언을 다시 한 번 새기게 된 순간이었습니다.

### 2. BERT 양자화 — 속도가 아니라 메모리

병목이 Synthesizer 라는 게 명확해졌지만, 그렇다고 BERT 양자화가 무의미한 건 아니었습니다. **속도가 아닌 메모리** 측면에서요.

BERT 양자화는 PyTorch 의 `torch.quantization.quantize_dynamic` 로 적용했습니다. Linear 레이어의 가중치를 INT8 로 압축하고, 추론 시점에 동적으로 양자화·역양자화를 수행하는 방식입니다.

```python
import torch
from torch.quantization import quantize_dynamic

quantized_bert = quantize_dynamic(
    bert_model,
    {torch.nn.Linear},
    dtype=torch.qint8,
)
```

결과를 비교해 보면:

| 구성 | 추론 시간 | RAM |
|---|---|---|
| PyTorch BERT FP32 | 4.796 s | +1,698 MB |
| PyTorch BERT Q8 | 4.536 s | **+368 MB** (−78 %) |

속도는 약 5 % 향상에 그쳤지만 (예상한 그대로), **메모리는 78 % 가 줄었습니다**. 한 서버에서 여러 프로그램을 돌리거나, 컨테이너 메모리 제한이 빡빡한 환경에서는 이 차이가 꽤 의미 있게 작용합니다.

#### Q4 vs Q8 — 어디까지 줄일 수 있나

여기서 한 단계 더 들어가 INT4 (Q4) 까지 시도해 봤습니다. 메모리를 더 줄일 수 있다면 좋으니까요.

| 구성 | BERT 크기 | RAM (1 화자) |
|---|---|---|
| FP32 | 1,157 MB | 1,599 MB |
| Q8 | 497 MB | 1,079 MB (−33 %) |
| Q4 | 394 MB | 958 MB (−40 %) |

다만 음질 검증을 해 보니 FP32 와 Q8 은 직접 청취 시 일관되게 구분하기 어려운 수준이었고, **Q4 는 대부분의 구간에서 유사하지만 문장 끝부분에서 미세한 차이가 들렸습니다.**

추가로 얻을 수 있는 메모리 이득 (Q8 → Q4 가 약 −7 %p) 이 청감 손실을 정당화하기엔 부족하다고 판단해, **기본값으로 Q8** 을 채택했습니다.

### 3. Synthesizer 는 왜 양자화 안 됐는가

다음 자연스러운 질문은 "그럼 Synthesizer 도 양자화하면 되지 않나?" 였습니다. 거기가 시간을 다 쓰니까요.

결론부터 말하면 **Synthesizer 양자화는 결국 적용하지 않았습니다.** 두 가지 방향을 시도해 봤는데 둘 다 별다른 효과가 없었거든요.

#### 1. FP16 캐스팅 (PyTorch) — Flow 레이어가 깨집니다

PyTorch 에서 Synthesizer 를 FP16 으로 캐스팅해 봤더니, Flow 레이어 안의 `rational_quadratic_spline` 이라는 함수가 정밀도 부족으로 깨지면서 일정 확률로 다음과 같은 assertion 이 터졌습니다.

```
AssertionError: discriminant < 0
```

이 함수는 입력을 일정한 규칙으로 변환해 출력을 만드는 **변환 함수** 입니다. VITS 추론 시에는 이 변환을 **거꾸로 (inverse pass)** 호출하는데, 그 과정에서 **이차방정식의 근의 공식** 을 사용합니다.

원본 SBV2 [`transforms.py`](https://github.com/litagin02/Style-Bert-VITS2/blob/66de777e06392c0f313600be03c43ef96658b244/style_bert_vits2/models/transforms.py#L167) 의 inverse 분기에서 발췌하면 다음과 같습니다.

```python
# 이차방정식 ax² + bx + c = 0 의 계수
a = (inputs - input_cumheights) * (
    input_derivatives + input_derivatives_plus_one - 2 * input_delta
) + input_heights * (input_delta - input_derivatives)
b = input_heights * input_derivatives - (inputs - input_cumheights) * (
    input_derivatives + input_derivatives_plus_one - 2 * input_delta
)
c = -input_delta * (inputs - input_cumheights)

discriminant = b.pow(2) - 4 * a * c
assert (discriminant >= 0).all()        # ← 여기서 깨집니다

root = (2 * c) / (-b - torch.sqrt(discriminant))
```

판별식 `b² - 4ac` 가 음수면 실근이 없어 변환이 정의되지 않으므로, 코드는 `assert` 로 그 가능성을 차단하고 있습니다. 수학적으로는 입력·weight 가 정상 범위 안에 있을 때 항상 `≥ 0` 이 보장되지만, **부동소수점에서는 이야기가 달라집니다.** FP16 으로 떨어지면 정밀도가 부족해 미세한 반올림 오차가 생기고, 그 결과 `discriminant` 가 음수가 되는 경우가 발생해 일정 확률로 assertion 이 터집니다.

#### 2. INT8 dynamic quantization (ONNX Runtime) — 양자화할 대상이 없습니다

다음으로는 ONNX Runtime 의 dynamic quantization 으로 시도해 봤습니다. 이쪽은 weight 만 INT8 로 저장하고 활성화는 FP32 그대로 흘러가는 방식이라, 적어도 spline 안의 산술이 깨지지는 않습니다.

다만 시도해 봤더니 또 다른 문제가 있었습니다. ONNX Runtime 의 dynamic quantization 은 **`MatMul` 연산만 양자화** 하는데, Synthesizer 는 `Conv1d` 위주라 사실상 양자화할 대상 자체가 거의 없습니다.

| 모델 | FP32 | Q8 | 변화 |
|---|---|---|---|
| BERT (DeBERTa 330M, MatMul 위주) | 1,159 MB | 544 MB | **−47 %** |
| Synthesizer (Conv1d 위주) | 239 MB | 239 MB | **0 %** |

실제로 Synthesizer 를 ONNX Q8 로 양자화해 봤지만 모델 파일 크기가 그대로였고, 추론 속도에도 변화가 거의 없었습니다.

추가로 Synthesizer 자체가 63 M 정도로 작아 — BERT 의 1/5 수준 — 어떻게 양자화하든 얻을 수 있는 메모리 이득이 BERT 만큼 크지 않다는 점도 있었습니다.

그러면 Synthesizer 속도는 어떻게 챙기나? 라는 질문이 자연스럽게 따라왔고, 그 답이 ONNX 였습니다.

### 4. ONNX Runtime 그래프 최적화

ONNX Runtime 은 모델을 로드할 때 **그래프 수준 최적화** 를 자동으로 적용합니다. 양자화 없이도 다음과 같은 변환을 거쳐 추론을 빠르게 만들어 줍니다.

- **Kernel fusion** — 연속된 여러 연산을 하나로 합칩니다. 예를 들어 `Conv → BatchNorm → Activation` 세 단계가 하나의 fused 커널이 되면, 중간 결과를 메모리에 쓰고 다시 읽는 비용이 사라져 메모리 대역폭이 절약됩니다.
- **Constant folding** — 입력에 관계없이 항상 같은 값을 내는 연산을 로드 시점에 미리 계산해 둡니다. 추론 시에는 미리 계산된 값을 그대로 사용합니다.
- **불필요한 노드 제거** — 사용되지 않거나 중복되거나 무의미한 연산 노드를 찾아 제거합니다.

여기에 더해, ONNX Runtime 은 [intra-op parallelism](https://onnxruntime.ai/docs/performance/tune-performance/threading.html) 으로 단일 연산을 여러 CPU 코어에 분산시킵니다. 동시 요청이 하나뿐이어도 CPU 전체를 활용할 수 있어, 단일 화자·실시간 추론 시나리오에 유리합니다.

#### 적용 결과 — CPU 배속

배속 = 오디오 길이 / 추론 시간 (값이 클수록 빠름).

| 구성 | short (1.7s) | medium (7.6s) | long (10.7s) | xlong (38.5s) |
|---|---|---|---|---|
| SBV2 PyTorch FP32 | 1.52× | 2.27× | 2.16× | 1.09× |
| SBV2 ONNX FP32 | 1.76× | 3.09× | 3.26× | 2.75× |
| **HayaKoe (Q8 BERT + FP32 ONNX)** | **2.50×** | **3.35×** | **3.33×** | **3.60×** |

xlong 텍스트 (38.5 초 분량) 기준으로, 원본 PyTorch 의 1.09× 가 HayaKoe 에서는 **3.60× 까지 올라갔습니다**. 실시간을 간신히 따라가던 수준에서, 동일 입력을 약 3 배 빠르게 처리할 수 있게 된 셈입니다.

#### 메모리 — BERT Q8 효과까지 합쳐서

| 구성 | RAM (1 화자) |
|---|---|
| SBV2 PyTorch FP32 | 5,122 MB |
| SBV2 ONNX FP32 | 2,967 MB |
| **HayaKoe (Q8 BERT + FP32 ONNX)** | **2,346 MB** (−54 %) |

ONNX 변환만으로도 RAM 이 약 42 % 줄고 (PyTorch overhead 가 빠지므로), 거기에 BERT Q8 양자화가 추가로 메모리를 잘라 최종 −54 % 가 됐습니다.

### 5. 메모리 89 MB 사건 — 측정의 함정

지금까지의 수치는 다 그럴듯해 보이지만, **이 수치를 신뢰할 수 있게 만들기까지** 한 번 크게 헤맸습니다.

처음 벤치마크를 짤 때는 단순했습니다. 한 Python 프로세스 안에서 PyTorch 모델 → ONNX FP32 → ONNX Q8 순서로 차례차례 로드하고, 각 시점에 RAM 을 측정해서 비교하는 식이었습니다.

```python
# 의사 코드
result["pytorch"]      = measure(load_pytorch_model)
result["onnx_fp32"]    = measure(load_onnx_fp32)
result["onnx_all_q8"]  = measure(load_onnx_q8)  # ← 여기서 89 MB 가 나옴
```

그런데 결과 JSON 을 보다가 이상한 값을 발견했습니다.

```
onnx_all_q8 RAM: 89 MB
```

**89 MB.** Q8 모델이 아무리 작다 해도, BERT INT8 + Synthesizer FP32 weight 만 합쳐도 거의 1 GB 가까이 되어야 합니다. 실제 같은 모델을 단독 프로세스로 띄워 보면 약 1,757 MB 가 나오는데, 단일 프로세스 측정에서는 89 MB 로 찍혔습니다.

#### 원인 추적 — Python 메모리 할당자

원인을 정확히 단정하긴 어려웠지만, 동작을 보고 추측한 가설은 — **이전 모델이 잡았던 메모리 영역을 다음 모델이 그대로 재사용한 게 아닐까** — 였습니다.

- 첫 모델 (PyTorch ~2,700 MB) 로드 → 프로세스 RSS 가 2.7 GB 까지 올라감
- 첫 모델을 `del` 로 해제 → Python 객체 그래프에서는 사라지지만, OS 입장에서는 여전히 프로세스가 그 영역을 들고 있는 듯
- 두 번째 모델 로드 → 새로 OS 에 메모리를 요구하지 않고, 앞에서 풀린 영역을 재사용한 것으로 추측됨
- `psutil` 로 본 RSS delta 는 두 번째 모델 로드 시 거의 0 → 작은 추가량인 89 MB 만 잡힘

즉 측정 자체는 "정확히 그 시점의 RSS 변화" 를 보긴 하지만, 우리가 알고 싶었던 "이 모델을 단독으로 쓸 때 얼마나 메모리를 먹는가" 라는 질문에는 잘못된 답을 주고 있었던 셈입니다.

`gc.collect()` 나 `del` 로 강제 해제도 시도해 봤지만 의미 있는 차이가 나오지 않았는데, 이것도 위 가설을 뒷받침하는 정황이었습니다.

#### 해결 — 프로세스 격리

결국 **각 config 를 독립된 subprocess 로 실행**하는 방식으로 측정 코드를 갈아엎었습니다. PyTorch 프로세스가 끝나면 OS 가 그 메모리를 회수하고, 그 다음 ONNX 프로세스는 깨끗한 상태에서 시작합니다.

```python
# 의사 코드
for config in ["pytorch", "onnx_fp32", "onnx_all_q8"]:
    result = subprocess.run(
        ["python", "measure_one.py", "--config", config],
        capture_output=True,
    )
    save_result(config, parse(result.stdout))
```

이렇게 격리하니 `onnx_all_q8` RAM 이 정상적으로 약 1,757 MB 로 측정됐고, 그 결과가 결국 위에서 보여 드린 "2,346 MB / -54 %" 수치의 근거가 됐습니다.

### Part 2 마치며

Part 2 에서는 5,122 MB · 1.09× 였던 모델이 어떻게 2,346 MB · 3.6× 로 다듬어졌는지를 정리했습니다.

요약하자면 — **BERT 는 Q8 양자화로 메모리 소모를 줄이고**, **Synthesizer 는 양자화 대신 ONNX 로 변환해 그래프 최적화 효과만 가져오고**, 마지막으로 **측정 환경 자체를 신뢰할 수 있게 만드는 데** 시간을 썼던 셈입니다.

이 정도면 일반적인 사용 시나리오에는 충분하지만, 실제로 다문장을 합성하다 보면 또 다른 디테일들이 드러나기 시작합니다. GPU 환경의 추가 가속, 다문장 BERT 배치 추론, 그리고 다문장 합성에서 사라지는 자연스러운 pause 같은 것들이요.

[Part 3 — 마지막 1% 까지: torch.compile, 배치 추론, Pause 복원, ARM64](/p/9a99dd816456c397/) 에서 이어집니다.
