---
title: "마지막 1% 까지 — torch.compile, 배치 추론, Pause 복원, ARM64 - hayakoe Part 3"
slug: 9a99dd816456c397
date: 2026-04-26T19:29:25+09:00
image: "cover.png"
categories:
    - AI
    - TTS
tags:
    - AI
    - TTS
    - ONNX
    - 최적화
    - PyTorch
    - 라즈베리파이
    - Python
---

hayakoe 시리즈의 세 번째 본편입니다. 이전 글은 [메모리는 절반으로, 속도는 1.5배 빠르게 한 이야기 - hayakoe Part 2](/p/137243ede98030f1/) 에서 보실 수 있습니다.

[Part 2](/p/137243ede98030f1/) 에서 큰 줄기 — BERT Q8 양자화 + Synthesizer ONNX 변환 — 로 메모리·속도를 꽤 개선했습니다. 다만 이 모델을 직접 운영해 보니, 큰 줄기와는 별개로 아쉽게 느껴졌던 부분들이 슬슬 눈에 들어왔습니다. 좀 더 쓰기 편한 형태로 다듬어 보면 좋겠다 싶은 작업들이었습니다.

Part 3 에서는 그렇게 손본 네 가지를 다룹니다.

1. **`torch.compile`** — GPU 추론 가속
2. **BERT GPU 유지 + 배치 추론** — 다문장 합성에서 의미 있는 차이
3. **자연스러운 pause 복원** — 다문장을 분할 합성하면 사라지는 구두점 뒤 쉼을 다시 살리기
4. **ARM64 빌드** — 라즈베리 파이 4B 에서도 돌아가게

### 1. `torch.compile` — GPU 추론 가속

PyTorch 2.0 부터 도입된 `torch.compile` 은 모델 그래프를 JIT 컴파일 (실행 시점에 동적으로 컴파일) 해 추가 속도 이득을 얻는 기능입니다. CUDA Graphs (반복되는 GPU 연산 시퀀스를 한 번에 묶어 재실행하는 기능) 로 GPU 호출 오버헤드를 줄이고, 가능한 곳은 여러 연산을 하나의 GPU 커널로 합친 형태 (fused 커널) 로 치환합니다.

hayakoe 에서는 `prepare()` 호출 시 device 가 CUDA 라면 자동으로 `torch.compile` 을 적용합니다 — 사용자 입장에서는 별도 설정 없이 그냥 GPU 모드만 켜면 됩니다.

| 백엔드 | 짧은 문장 | 중간 문장 | 긴 문장 |
|---|---|---|---|
| PyTorch (CUDA) | 7.3× | 16.3× | 13.6× |
| `torch.compile` | 7.4× | 17.2× | **15.4×** |
| 향상폭 | +1 % | +6 % | **+13 %** |

긴 문장에서 향상폭이 큰 이유는, 텍스트가 길수록 Synthesizer 가 호출하는 Conv 커널 수가 많아지고 그만큼 런치 오버헤드도 누적되기 때문입니다. CUDA Graphs 가 그 오버헤드를 한 번에 흡수해 주는 셈입니다.

다만 이 효과를 얻으려면 **워밍업** 이 필요합니다. CUDA Graphs 가 그래프를 캡처하고 컴파일하는 데 시간이 걸려서, 첫 몇 번의 호출은 오히려 더 느리게 나옵니다. hayakoe 는 `prepare(warmup=True)` 옵션으로 8 회 정도 더미 추론을 돌려, 사용자가 첫 요청에서 컴파일 비용을 보지 않도록 합니다.

### 2. BERT GPU 유지 + 배치 추론

GPU 경로의 효율을 갉아먹던 두 가지 — **불필요한 GPU↔CPU 왕복** 과 **문장별 BERT 개별 호출** — 도 같이 손봤습니다.

#### `.cpu()` 제거 — GPU 텐서 유지

원본 SBV2 의 [BERT feature 추출 코드](https://github.com/litagin02/Style-Bert-VITS2/blob/66de777e06392c0f313600be03c43ef96658b244/style_bert_vits2/nlp/japanese/bert_feature.py#L61) 에 다음과 같은 부분이 있었습니다.

```python
# 원본 SBV2 (style_bert_vits2/nlp/japanese/bert_feature.py)
res = torch.cat(res["hidden_states"][-3:-2], -1)[0].cpu()
```

BERT 를 GPU 에서 forward 한 뒤 출력에 `.cpu()` 를 호출해 매번 CPU 로 내리고 있었는데, 이 출력은 곧바로 Synthesizer (역시 GPU) 에 전달되니 다시 GPU 로 올려야 합니다. 결과적으로 문장마다 **GPU → CPU → GPU 왕복** 이 발생하고, 이 왕복 자체가 작은 병목이 됩니다.

원본 코드를 다음처럼 바꿔서 BERT 출력이 GPU 텐서 그대로 흐르도록 했습니다.

```python
# hayakoe
res = torch.cat(res["hidden_states"][-3:-2], -1)[0].float()  # GPU 유지
```

`.cpu()` 대신 `.float()` 만 호출하는 이유는 dtype 통일 (FP16 BERT 와 FP32 Synthesizer 사이 캐스팅) 때문이고, 자세한 사정은 [Part 2 의 BERT 양자화 / FP16 캐스팅 부분](/p/137243ede98030f1/) 에서 다뤘습니다.

추가로 BERT 모델 자체를 **글로벌 싱글턴** 으로 관리해, 화자가 여러 명이어도 BERT 는 한 번만 GPU 에 올라가고 모든 화자가 그 인스턴스를 공유하도록 했습니다.

#### 다문장 BERT 배치화

hayakoe 는 prosody (운율) 안정성을 위해 입력 텍스트를 구두점 기준으로 분할해 문장 단위로 합성합니다. 그러다 보니 BERT 가 문장 수만큼 반복 호출되는 구조가 자연스럽게 따라옵니다.

GPU 에서는 연산을 호출할 때마다 일정한 고정 비용 (kernel launch overhead) 이 붙기 때문에, 문장이 짧으면 실제 연산 시간보다 이 호출 비용이 더 큰 비효율이 누적됩니다. 다행히 BERT (DeBERTa) 는 HuggingFace Transformer 라 batch 입력을 기본 지원하므로, **모든 문장을 하나의 배치로 묶어 BERT 를 1 회만 forward** 하도록 처리했습니다.

| 문장 수 | 순차 | 배치 | 속도 향상 |
|---|---|---|---|
| 2 | 0.447 s | 0.364 s | **1.23×** |
| 4 | 0.812 s | 0.566 s | **1.43×** |
| 8 | 1.598 s | 1.121 s | **1.43×** |
| 16 | 2.972 s | 2.264 s | **1.31×** |

문장 수가 늘수록 +23 % ~ +43 % 의 속도 향상이 안정적으로 나타납니다. 메모리 차이는 1.3 MB 이내라 사실상 동일했고요.

흥미로운 점은 같은 실험을 CPU (ONNX) 에서 반복하면 효과가 거의 없다는 점이었습니다. 측정해 보면 +1 % ~ −10 % 사이의 노이즈 수준 차이만 나옵니다.

CPU 에서는 큰 효과가 없지만 손해도 거의 없으니, GPU·CPU 양쪽에서 같은 코드 경로로 동작하도록 배치화는 그대로 켜 두는 쪽으로 정리했습니다.

### 3. 자연스러운 pause 복원

이번 Part 에서 가장 디테일이 많은 부분입니다.

#### 분할 합성의 부작용

위 섹션에서 말했듯, hayakoe 는 prosody (운율) 안정성을 위해 다문장을 구두점 기준으로 쪼갠 뒤 각 문장을 따로 합성합니다. 긴 텍스트를 통째로 합성하면 억양이 뭉개지거나 불안정해지는 경향이 있어서, 안정성과 자연스러움을 위해 도입한 구조입니다.

하지만 이 분할에는 부작용이 하나 있습니다. **문장 사이의 자연스러운 쉼 (pause) 이 사라진다** 는 것이었습니다.

원본 SBV2 는 통문 합성에서 `.`, `!`, `?` 같은 구두점 뒤에 자연스러운 쉼을 만들어 줍니다. 그런데 문장 단위로 분할하면 각 문장이 구두점에서 끝나고 다음 문장이 처음부터 시작되므로, 구두점 뒤의 쉼이 함께 사라집니다. 초기 구현에서는 문장 사이에 고정 80 ms 무음을 삽입해 봤는데, 실제로 자연스러운 쉼은 0.3 ~ 0.6 초 수준이라 80 ms 는 너무 짧고 결과적으로 "숨 돌릴 틈이 없는" 부자연스러운 발화가 만들어졌습니다.

#### 원본 SBV2 는 pause 를 어떻게 만들었나

원본 SBV2 가 통문 합성에서 자연스러운 쉼을 만드는 원리를 추적해 봤습니다. 결론은 의외로 단순했습니다 — **Duration Predictor 가 구두점 음소의 frame 수를 예측하는 부수 효과** 였습니다.

Duration Predictor 는 원래 "각 음소를 몇 프레임 동안 발음할지" 를 예측하는 모듈입니다. "안" 은 5 프레임, "녕" 은 4 프레임 같은 식으로요. 그런데 `.`, `!`, `?` 같은 구두점도 음소 시퀀스에 포함되어 있고, Duration Predictor 는 이 구두점 음소에도 frame 수를 예측합니다. 예측된 frame 수가 곧 그 위치에서의 쉼 길이가 되는 것입니다.

분할 합성에서는 구두점 위치에서 합성이 끊기므로 이 정보가 그대로 폐기되고 있었습니다.

#### 해결 — Duration Predictor 만 따로 실행

문제와 원인이 모두 분명해졌으니 해결도 자연스럽게 따라왔습니다.

핵심 아이디어는 간단합니다. **분할 전 원문 텍스트를 TextEncoder + Duration Predictor 까지만 통과시켜, 구두점 위치의 frame 수를 미리 얻어 두는 것** 입니다. Flow 와 Decoder (실제 오디오를 만드는 부분) 는 건너뜁니다.

```
전체 텍스트 (분할 전 원문)
  │
  ├─ TextEncoder (G2P → 음소열 → 임베딩)
  │
  ├─ Duration Predictor (음소별 frame 수 예측)
  │     └─ 구두점 위치의 frame 수만 추출
  │
  └─ pause 시간 계산
        frames × hop_length / sample_rate = 초
```

Synthesizer 전체 pass 비용의 대부분은 Flow + Decoder 에 있으므로 ([Part 2 의 병목 측정](/p/137243ede98030f1/) 참고), Duration Predictor 까지만 실행하는 비용은 전체 합성 대비 매우 낮습니다.

`hop_length = 512`, `sample_rate = 44100` 인 hayakoe 기본 설정에서 1 프레임은 약 11.6 ms 에 해당하므로, 구두점 + 인접 blank token 의 합산 frame 수가 35 라면:

```
35 × 512 / 44100 ≈ 0.41 초
```

이런 식으로 각 문장 경계마다 자연스러운 pause 시간을 얻을 수 있습니다.

#### ONNX 지원 — Duration Predictor 만 따로 export

PyTorch 경로에서는 모델 내부 모듈을 개별 호출할 수 있어 Duration Predictor 만 골라 실행하면 됩니다. 그런데 `synthesizer.onnx` 는 Synthesizer 전체를 하나의 종단 간 그래프로 내보낸 형태라, 중간에서 Duration Predictor 출력만 꺼내 쓰는 게 불가능했습니다.

이를 해결하기 위해 **TextEncoder + Duration Predictor 만 포함하는 별도 ONNX 모델** 을 추가로 export 했습니다.

- 산출물: `duration_predictor.onnx` (~30 MB, FP32)
- ONNX Runtime 위에서 실행
- 기존 배포 모델에 이 파일이 없으면 조용히 80 ms fallback 으로 동작 (하위 호환)

#### 결과

동일 텍스트에 대해 자동 예측된 문장 경계 pause:

| 백엔드 | pause 범위 |
|---|---|
| GPU (PyTorch) | 0.41 s ~ 0.55 s |
| CPU (ONNX) | 0.38 s ~ 0.57 s |

두 백엔드의 차이는 SDP (Stochastic Duration Predictor) 의 확률 샘플링 특성상 발생하는 자연 변동폭 안에 들어옵니다. 즉 ONNX 변환으로 인한 품질 손실은 사실상 없습니다.

다문장 샘플을 직접 들어보면 차이가 꽤 명확합니다. 80 ms 고정 무음이었던 Before 는 문장이 거의 붙어 들리는데, Duration Predictor 가 예측한 pause 가 들어간 After 는 사람 발화의 호흡 흐름에 가까워집니다.

### 4. ARM64 빌드 — 라즈베리 파이 4B

마지막으로, hayakoe 는 x86_64 뿐 아니라 **aarch64 (ARM64) Linux 에서도 동일한 코드로 동작** 하게 만들었습니다.

이게 가능한 건 두 가지 조건이 갖춰져 있기 때문입니다.

- **ONNX Runtime** 이 aarch64 빌드를 공식 제공
- **`pyopenjtalk`** 의 자체 fork ([`lemon-pyopenjtalk-prebuilt`](https://github.com/LemonDoubleHQ/lemon-pyopenjtalk-prebuilt), [관련 글](https://blog.lemondouble.com/p/pyopenjtalk-prebuilt/)) 가 aarch64 wheel 을 빌드하고, 사전을 패키지에 함께 포함

두 번째 조건은 [서론](/p/ea78fbb9b78c38ac/) 에서 언급한 "`openjtalk` 외부 다운로드 의존" 문제를 해결하면서 자연스럽게 따라온 결과물이기도 합니다.

#### Raspberry Pi 4B 실측

Raspberry Pi 4B (Linux 6.8, aarch64, ONNX Runtime 1.23.2) 에서 측정한 결과:

| 텍스트 | 추론 시간 | 배속 |
|---|---|---|
| 짧음 | 3.169 s | 0.3× |
| 중간 | 13.042 s | 0.3× |
| 김 | 35.119 s | 0.3× |

실시간의 약 1/3 수준이라 대화형 용도로는 부족합니다. 다만 ARM 보드에서 돌아간다는 것 자체에 의미가 있다고 생각합니다 — 오프라인 배치 합성이라거나, 클러스터의 ARM 노드에서 비동기 작업으로 돌리는 시나리오에는 충분히 활용 가능합니다.

> Apple Silicon (macOS) 에서도 동작할 것으로 예상하지만, 테스트 장비가 없어 확인하지는 못했습니다.

### Part 3 마치며

Part 3 에서는 큰 최적화 줄기 외에 **잔디 다듬기 수준의 디테일 네 가지** 를 정리했습니다.

`torch.compile` 로 긴 문장에서 +13 % 를 더 챙기고, BERT GPU 유지·배치 추론으로 GPU 경로의 비효율을 잡고, Duration Predictor 를 따로 떼어 다문장 분할 합성의 자연스러움을 회복하고, 마지막으로 라즈베리 파이 4B 까지 동작 범위를 넓힌 작업이었습니다.

이걸로 모델·추론 측면의 최적화는 사실상 마무리되었습니다. 마지막 Part 에서는 다른 사람도 쉽게 가져다 쓸 수 있는 라이브러리로 어떻게 패키징했는지 — API 설계, multi-source, thread-safe 싱글톤 서빙, FastAPI / Docker 패턴 — 를 다룰 예정입니다.

[Part 4 — 라이브러리로 패키징하면서 고민한 것들](/p/b7791b4f6c0818da/) 에서 이어집니다.
