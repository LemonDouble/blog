---
title: "라이브러리로 패키징하면서 고민한 것들 - hayakoe Part 4"
slug: b7791b4f6c0818da
date: 2026-04-26T23:23:19+09:00
image: "cover.png"
categories:
    - AI
    - TTS
tags:
    - AI
    - TTS
    - Python
    - 라이브러리
    - FastAPI
    - Docker
    - API 설계
---

hayakoe 시리즈의 네 번째이자 마지막 본편입니다. 이전 글은 [마지막 1% 까지 — torch.compile, 배치 추론, Pause 복원, ARM64 - hayakoe Part 3](/p/9a99dd816456c397/) 에서 보실 수 있습니다.

[Part 1](/p/4c2a6225a737f531/) 부터 [Part 3](/p/9a99dd816456c397/) 까지는 모델·추론 측면의 작업이었다면, Part 4 는 **그 결과물을 다른 사람도 (그리고 미래의 내가도) 쉽게 가져다 쓸 수 있게 패키징하는 과정** 에서 했던 고민들을 정리합니다.

크게 네 가지로 나눠 봤습니다.

1. **API 설계** — `from_pretrained` 같은 친숙한 호출 + 단계별 체이닝
2. **Source 추상화** — `hf://` / `s3://` / `file://` 를 한 인스턴스에서 섞어 쓰기
3. **Thread-safe 싱글톤 서빙** — FastAPI 동기·비동기 모두 안전하게
4. **Docker 빌드 패턴** — 빌드 시점에 모델 다 받아두고 런타임은 네트워크 없이

### 1. API 설계 — 친숙한 호출 + 단계별 체이닝

라이브러리를 만들기로 결정한 시점에 가장 먼저 정한 건 "어떤 API 로 노출할 것인가" 였습니다. 결론적으로는 두 가지를 합친 형태로 갔습니다 — HuggingFace `transformers` 의 `from_pretrained` 같은 **이름으로 모델을 로드하는 친숙한 호출**, 그리고 비용이 다른 단계를 분리하는 **체이닝**.

먼저 친숙한 호출 쪽을 보면, `transformers` 는 이런 식으로 씁니다 — 모델 이름만 넘기면 자동으로 다운로드 / 캐시 / 로드까지 해 줍니다.

```python
from transformers import AutoModel

model = AutoModel.from_pretrained("bert-base-uncased")
```

hayakoe 도 화자 이름만 넘기면 같은 동작이 일어나도록 잡았습니다. 거기에 더해, 비용이 다른 단계를 분리하기 위해 체이닝을 더했습니다.

```python
from hayakoe import TTS

tts = TTS().load("jvnv-F1-jp").prepare()
tts.speakers["jvnv-F1-jp"].generate("こんにちは").save("output.wav")
```

`prepare()` 와 실제 합성 (`generate()`) 을 따로 둔 이유는 단순합니다 — **무거운 비용은 서버가 요청을 받기 전에 미리 치러 두고**, 실제 요청은 합성만 해서 즉시 응답으로 돌려주기 위해서입니다. 모델 로드와 (CUDA 라면) `torch.compile` 비용까지 `prepare()` 에 모아 두면, 첫 요청부터 안정적으로 빠르게 처리됩니다.

#### 화자별 메모리 분리

라이브러리 설계 단계에서 신경 쓴 부분 하나가 **다화자 서빙 시의 메모리 효율** 이었습니다. 같은 BERT 를 화자마다 따로 로드하면 N 명을 서빙할 때 BERT 가 N 번 올라가서, 서버 메모리가 빠르게 차오르거든요.

[Part 2](/p/137243ede98030f1/) 에서 본 것처럼 BERT (DeBERTa) 가 모델 전체의 약 84 % 를 차지하므로, 이걸 화자 간 공유하는 건 메모리 효율에 직결됩니다.

```
TTS (엔진 — 공유 리소스)
├── BERT (DeBERTa, ~329M)   ← 1회만 로드, 모든 화자 공유
│
├── speakers["jvnv-F1-jp"]  → Synthesizer + style vectors (~250MB)
├── speakers["jvnv-F2-jp"]  → ...
└── ...
```

이 구조 덕에 화자가 늘어도 메모리가 선형으로 폭증하지 않습니다. 화자 한 명이 추가될 때마다 늘어나는 건 약 250 ~ 300 MB (CPU RAM 기준 약 300 ~ 400 MB) 정도로, 한 서버에서 여러 화자를 돌리는 시나리오가 현실적이게 됩니다.

### 2. Source 추상화 — multi-source URI 라우팅

라이브러리를 굴리다 보면 화자 모델이 **어디에 있는가** 가 상황마다 달라집니다.

- 공식 기본 화자는 HuggingFace 공개 레포
- 직접 학습한 사용자는 private HF 레포
- S3 등의 인프라가 이미 있다면 S3 (또는 R2 · MinIO 같은 S3 호환 스토리지)
- 개발 중인 모델 / 따로 백업을 원하면 로컬 디렉토리

매 소스마다 다운로드 코드를 분기 처리하면 엔진 본체가 비대해지고, 각자 다른 캐시 경로를 쓰게 되어 관리가 어려워집니다. 그래서 모든 소스를 **공통 인터페이스 뒤에 숨기고**, 사용자는 URI 만 바꿔주면 같은 API 로 동작하도록 추상화했습니다.

```python
class Source(Protocol):
    def fetch(self, prefix: str) -> Path:
        """prefix/ 아래 모든 파일을 캐시에 받고 로컬 경로를 반환."""
        ...

    def upload(self, prefix: str, local_dir: Path) -> None:
        """local_dir 내용을 prefix/ 아래로 업로드 (배포용)."""
        ...
```

URI 스킴으로 자동 라우팅됩니다.

| URI 스킴 | 구현 | 동작 |
|---|---|---|
| `hf://user/repo` | `HFSource` | `huggingface_hub.snapshot_download()` 로 다운로드 |
| `s3://bucket/prefix` | `S3Source` | `boto3` 기반. `AWS_ENDPOINT_URL_S3` 로 S3 호환 엔드포인트 (R2 · MinIO 등) 지원 |
| `file:///abs/path` 또는 `/abs/path` | `LocalSource` | 로컬 디렉토리 그대로 사용, 다운로드 없음 |

한 인스턴스에서 여러 소스를 섞어 써도 됩니다.

```python
tts = (
    TTS(device="cuda")
    .load("jvnv-F1-jp")                                  # 공식 HF
    .load("my-voice", source="hf://me/private-voices")   # private HF
    .load("client-a", source="s3://tts-prod/voices")     # S3
    .load("dev-voice", source="file:///mnt/experiments") # 로컬
    .prepare()
)
```

캐시는 모든 소스가 같은 루트 (`HAYAKOE_CACHE` 환경변수 또는 기본 `$CWD/hayakoe_cache`) 를 공유하므로, 한 디렉토리만 관리하면 됩니다. 새로운 스토리지를 지원해야 할 일이 생기면 `Source` 프로토콜만 구현하면 끝나는 구조입니다.

### 3. Thread-safe 싱글톤 서빙 — FastAPI 동기·비동기

이쪽은 라이브러리를 실제 서버로 운영해 보면서 가장 많이 다듬은 부분입니다.

#### 싱글톤이 아니면 서비스 자체가 안 됨

처음에는 단순하게 "요청마다 `TTS()` 인스턴스를 만들어서 처리" 같은 패턴도 가능할까 싶었는데, 이건 사실상 불가능합니다.

GPU 모드에서는 `prepare()` 가 `torch.compile` 을 돌리기 때문에 첫 컴파일에 수십 초가 걸립니다. 매 요청마다 이걸 다시 돌리면 응답이 안 끝나죠. CPU 모드라도 모델 로드만 몇 초 걸리니까 마찬가지입니다.

그래서 hayakoe 의 권장 패턴은 **앱 수명 동안 TTS 인스턴스 하나를 유지** 하는 것입니다. FastAPI 라면 lifespan 에서 한 번 빌드해서 `app.state` 에 붙이는 식.

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    tts = TTS(device="cuda")
    for name in SPEAKERS:
        tts.load(name)
    tts.prepare(warmup=True)   # torch.compile 까지 미리
    app.state.tts = tts
    yield
```

`warmup=True` 는 더미 추론을 8 회 정도 미리 돌려서, 첫 실제 요청이 컴파일 비용을 떠안지 않게 합니다. 서빙 환경에서는 거의 항상 켜는 옵션입니다.

#### 동시 요청 안전성 — per-speaker `threading.Lock`

서버에서는 동시에 여러 요청이 들어올 수 있습니다. 같은 화자에 대한 동시 합성 요청은 GPU/CPU 자원을 공유해야 하므로 어차피 직렬로 처리되어야 하는데, 이걸 매 호출마다 사용자가 락을 걸어주는 건 번거롭습니다.

그래서 `Speaker` 객체 안에 `threading.Lock` 을 들고 있어, 같은 화자에 대한 동시 호출은 자동으로 직렬화됩니다.

- **같은 화자 동시 호출** → 직렬 (락 대기)
- **다른 화자 동시 호출** → 병렬 (각 화자가 별도 락)

여러 화자를 미리 `load` 해 두면, 화자 간 병렬 처리는 자동으로 됩니다.

#### 비동기 핸들러는 `agenerate` / `astream`

FastAPI async 핸들러에서 동기 `generate()` 를 그대로 부르면 합성 시간 동안 이벤트 루프가 멈춥니다. 그래서 async 래퍼도 같이 제공하고, 내부적으로는 `asyncio.to_thread` 로 워커 스레드에 오프로드합니다.

| 동기 | 비동기 |
|---|---|
| `speaker.generate(text)` | `await speaker.agenerate(text)` |
| `speaker.stream(text)` | `async for chunk in speaker.astream(text)` |

한 가지 주의할 점은, `astream` 은 제너레이터가 살아있는 동안 per-speaker lock 을 잡고 있다는 것입니다. `async for` 를 중간에 끊으면 다른 요청이 오래 대기할 수 있어, FastAPI 의 `StreamingResponse` 와 함께 쓰는 게 안전합니다 (클라이언트 연결이 끊기면 자동으로 제너레이터를 닫아줍니다).

### 4. Docker 빌드 패턴 — 빌드 시점에 모델까지 굽기

마지막으로, 운영 환경에서 자주 마주치는 두 가지 요구를 위해 Docker 빌드 패턴도 다듬었습니다.

- **오프라인 환경 대응** — 한 번 구운 Docker 이미지만 있으면 외부 네트워크나 외부 의존성 없이 바로 추론을 시작할 수 있어야 합니다. 런타임에 HuggingFace · S3 등에서 모델을 내려받는 단계가 끼면 첫 요청이 너무 늦어지거나, 네트워크가 막혔을 때 배포 자체가 깨질 수 있거든요.
- **자격 증명 노출 회피** — HF 토큰이나 S3 키를 런타임 환경에 두고 싶지 않은 경우

해법은 단순합니다. **빌드 시점에 모델을 이미지 안에 다 받아두고**, 런타임 컨테이너는 캐시에서 즉시 로드만 하도록 분리하는 것.

이를 위해 `TTS.pre_download()` 라는, **모델을 캐시에만 채우고 실제 초기화는 안 하는** 메서드를 별도로 제공합니다. 핵심은 이게 **GPU 없이도 동작** 한다는 점 — GitHub Actions 같은 CPU 전용 CI 러너에서도 GPU용 이미지를 빌드할 수 있습니다.

```dockerfile
# syntax=docker/dockerfile:1.7
FROM python:3.12-slim-bookworm AS builder

# ... 의존성 설치 ...

ENV HAYAKOE_CACHE=/server/hayakoe_cache
RUN --mount=type=secret,id=hf_token,env=HUGGINGFACE_TOKEN \
    python -c "\
import os; from hayakoe import TTS; \
tts = TTS(hf_token=os.environ.get('HUGGINGFACE_TOKEN')); \
tts.load('my-voice', source='hf://me/my-voices'); \
tts.pre_download(device='cuda')"

FROM python:3.12-slim-bookworm AS prod
COPY --from=builder /server /server
ENV HAYAKOE_CACHE=/server/hayakoe_cache
ENTRYPOINT ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "80"]
```

BuildKit secret (`--mount=type=secret`) 으로 HF 토큰을 주입하면 토큰이 이미지 레이어에 남지 않습니다. 자세한 워크플로우 (GHCR push, GitHub Actions secret 주입 등) 는 hayakoe docs 의 [Docker 이미지](https://lemondouble.github.io/hayakoe/deploy/docker) 페이지에서 다뤘습니다.

CPU 전용 이미지로 만들면 PyTorch 가 빠지고 ONNX Runtime 만 들어가, 이미지 크기가 GB 단위에서 수백 MB 까지 줄어듭니다.

### Part 4 마치며 — 그리고 시리즈를 마치며

Part 4 에서는 학습된 모델이 라이브러리로 패키징되기까지 거친 네 가지 결정 — API 체이닝, 소스 추상화, thread-safe 싱글톤, Docker 빌드 패턴 — 을 정리했습니다.

여기까지가 hayakoe 4 부작 시리즈입니다. 처음에는 가볍게 시작했는데 — "잘 쓰던 TTS 라이브러리가 슬슬 거슬려서 한번 갈아엎어 보자" — 정신 차려 보니 모델 비교, 양자화, ONNX 변환, GPU 가속, 라이브러리 설계, Docker 빌드까지 다루고 있었습니다.

#### 시리즈 다시 돌아보기

- **[서론](/p/ea78fbb9b78c38ac/)** — 왜 갈아엎기로 했나, 결과물은 무엇인가
- **[Part 1](/p/4c2a6225a737f531/)** — 2년만에 다시 TTS 모델 이것저것 보다 또 다시 VITS로 정착한 이야기
- **[Part 2](/p/137243ede98030f1/)** — 메모리는 절반으로, 속도는 1.5 배 빠르게 한 이야기
- **[Part 3](/p/9a99dd816456c397/)** — 마지막 1 % 까지: torch.compile, 배치 추론, Pause 복원, ARM64
- **Part 4 (이 글)** — 라이브러리로 패키징하면서 고민한 것들

#### 결과물

- **Repository**: [github.com/LemonDouble/hayakoe](https://github.com/LemonDouble/hayakoe)
- **Documentation**: [lemondouble.github.io/hayakoe](https://lemondouble.github.io/hayakoe/)
- **PyPI**: `pip install hayakoe`

#### 그리고

처음 동기였던 "내가 쓰기 편한 TTS" 는 이제 잘 동작합니다. 알람·브리핑 용도로 매일 쓰고 있고, 더 이상 외부 다운로드 서버가 죽을까 걱정하지 않아도 되며, Python 버전업도 막힘없이 따라가고 있습니다.

남은 과제가 있다면 한국어 지원입니다. 현재 hayakoe 는 JP-Extra 모델 기반이라 한국어 음성은 합성이 안 되는데, 한국어 BERT (klue/roberta-large 등) + 한국어 G2P + 공개 데이터셋으로 사전학습을 다시 돌리는 작업이 필요합니다. 시간이 들고 GPU 도 많이 필요해서... 언젠간 시간이 난다면 해 보고 싶습니다.

긴 글 읽어 주셔서 감사합니다.
