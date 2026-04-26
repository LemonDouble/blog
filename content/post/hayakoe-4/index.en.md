---
title: "What I Considered When Packaging It as a Library — hayakoe Part 4"
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
    - Library
    - FastAPI
    - Docker
    - API Design
---

_Note: I'm based in Korea, so some context here is Korea-specific._

This is the fourth and final main entry of the hayakoe series. You can read the previous post at [Squeezing Out the Last 1% — torch.compile, Batch Inference, Pause Restoration, ARM64 - hayakoe Part 3](/en/p/9a99dd816456c397/).

If [Part 1](/en/p/4c2a6225a737f531/) through [Part 3](/en/p/9a99dd816456c397/) were about the model and inference side, Part 4 organizes the considerations I had **while packaging that result so other people (and future me) can easily reuse it**.

I've broken it down into four areas.

1. **API design** — a familiar `from_pretrained`-style call + step-by-step chaining
2. **Source abstraction** — mixing `hf://` / `s3://` / `file://` in a single instance
3. **Thread-safe singleton serving** — safe with both sync and async FastAPI
4. **Docker build pattern** — pull all models at build time, run with no network at runtime

### 1. API Design — Familiar Calls + Step-by-Step Chaining

The first thing I decided when I committed to building a library was "what API should this expose?" In the end, I went with a hybrid of two ideas — a **familiar load-by-name call** like HuggingFace `transformers`' `from_pretrained`, plus **chaining** that separates steps with different costs.

Looking at the familiar call side first, this is how `transformers` is used — pass a model name and it auto-downloads / caches / loads.

```python
from transformers import AutoModel

model = AutoModel.from_pretrained("bert-base-uncased")
```

I designed hayakoe so passing a speaker name triggers the same flow. On top of that, I added chaining to separate steps with different costs.

```python
from hayakoe import TTS

tts = TTS().load("jvnv-F1-jp").prepare()
tts.speakers["jvnv-F1-jp"].generate("こんにちは").save("output.wav")
```

The reason I split `prepare()` from the actual synthesis (`generate()`) is simple — **pay the heavy cost before the server starts taking requests**, so actual requests only do synthesis and return immediately. By collecting the model load and (on CUDA) `torch.compile` cost into `prepare()`, even the first request is reliably fast.

#### Per-Speaker Memory Separation

One thing I cared about during library design was **memory efficiency for multi-speaker serving**. If the same BERT is loaded separately per speaker, BERT gets loaded N times for N speakers and server memory fills up fast.

As we saw in [Part 2](/en/p/137243ede98030f1/), BERT (DeBERTa) accounts for about 84% of the entire model, so sharing it across speakers directly translates into memory efficiency.

```
TTS (engine — shared resource)
├── BERT (DeBERTa, ~329M)   ← loaded once, shared by all speakers
│
├── speakers["jvnv-F1-jp"]  → Synthesizer + style vectors (~250MB)
├── speakers["jvnv-F2-jp"]  → ...
└── ...
```

Thanks to this structure, memory doesn't blow up linearly when speakers are added. Each additional speaker adds only about 250–300 MB (or ~300–400 MB on CPU RAM), which makes running multiple speakers per server a realistic scenario.

### 2. Source Abstraction — Multi-Source URI Routing

When you actually run a library, **where the speaker model lives** changes by situation.

- Official default speakers live in a public HuggingFace repo
- Users who trained their own use a private HF repo
- If S3-style infrastructure is already in place, S3 (or S3-compatible storage like R2 / MinIO)
- For models still in development or wanted as a backup, a local directory

If you branch the download code per source, the engine itself bloats and each source ends up with a different cache path, which gets hard to manage. So I **hid every source behind a common interface**, and abstracted it so the user only changes the URI while the API stays the same.

```python
class Source(Protocol):
    def fetch(self, prefix: str) -> Path:
        """Download all files under prefix/ to the cache and return the local path."""
        ...

    def upload(self, prefix: str, local_dir: Path) -> None:
        """Upload contents of local_dir under prefix/ (for distribution)."""
        ...
```

Routing is automatic by URI scheme.

| URI scheme | Implementation | Behavior |
|---|---|---|
| `hf://user/repo` | `HFSource` | Downloads via `huggingface_hub.snapshot_download()` |
| `s3://bucket/prefix` | `S3Source` | `boto3`-based. Supports S3-compatible endpoints (R2, MinIO, etc.) via `AWS_ENDPOINT_URL_S3` |
| `file:///abs/path` or `/abs/path` | `LocalSource` | Uses the local directory as is, no download |

You can mix multiple sources in a single instance.

```python
tts = (
    TTS(device="cuda")
    .load("jvnv-F1-jp")                                  # official HF
    .load("my-voice", source="hf://me/private-voices")   # private HF
    .load("client-a", source="s3://tts-prod/voices")     # S3
    .load("dev-voice", source="file:///mnt/experiments") # local
    .prepare()
)
```

All sources share the same cache root (`HAYAKOE_CACHE` env var or default `$CWD/hayakoe_cache`), so you only manage one directory. If you ever need to support a new storage backend, you just implement the `Source` protocol and you're done.

### 3. Thread-Safe Singleton Serving — FastAPI Sync/Async

This is the part I polished the most while actually running the library as a server.

#### Without a Singleton, the Service Doesn't Work

At first I wondered whether a simple "create a `TTS()` instance per request" pattern would work, but it's basically impossible.

In GPU mode, `prepare()` runs `torch.compile`, so the first compile takes tens of seconds. Repeating that per request means the response never finishes. Even in CPU mode, model loading alone takes a few seconds, so it's the same.

So hayakoe's recommended pattern is to **keep one TTS instance for the lifetime of the app**. With FastAPI, that means building it once in lifespan and attaching it to `app.state`.

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    tts = TTS(device="cuda")
    for name in SPEAKERS:
        tts.load(name)
    tts.prepare(warmup=True)   # including torch.compile
    app.state.tts = tts
    yield
```

`warmup=True` runs about 8 dummy inferences in advance so the first real request doesn't pay the compile cost. In serving environments, it's almost always on.

#### Concurrent Request Safety — Per-Speaker `threading.Lock`

A server can receive many requests at once. Concurrent synthesis requests for the same speaker have to share the GPU/CPU resources, so they end up serialized anyway, and forcing the user to lock per call is annoying.

So each `Speaker` object holds a `threading.Lock`, and concurrent calls for the same speaker are serialized automatically.

- **Same speaker, concurrent calls** → serial (lock wait)
- **Different speakers, concurrent calls** → parallel (each speaker has its own lock)

If you `load` multiple speakers in advance, parallelism across speakers happens automatically.

#### Async Handlers Use `agenerate` / `astream`

If you call sync `generate()` directly from a FastAPI async handler, the event loop blocks during synthesis. So I provide async wrappers too, which internally offload to a worker thread via `asyncio.to_thread`.

| Sync | Async |
|---|---|
| `speaker.generate(text)` | `await speaker.agenerate(text)` |
| `speaker.stream(text)` | `async for chunk in speaker.astream(text)` |

One thing to watch: `astream` holds the per-speaker lock for as long as the generator is alive. Cutting `async for` short can leave other requests waiting, so it's safe to use it together with FastAPI's `StreamingResponse` (which automatically closes the generator when the client disconnects).

### 4. Docker Build Pattern — Bake Models In at Build Time

Finally, I refined the Docker build pattern around two requirements I run into often in production.

- **Offline-environment support** — once you have a built Docker image, it should be able to start inference immediately without external network or external dependencies. If model downloads from HuggingFace / S3 are required at runtime, the first request is too slow, and a network block can break deployment entirely.
- **Avoid exposing credentials** — if you don't want HF tokens or S3 keys sitting in the runtime environment

The fix is straightforward. **Pull all models into the image at build time**, and have the runtime container only load from cache.

For this, hayakoe provides a separate `TTS.pre_download()` method that **only fills the cache without actually initializing**. The key point is that this **works without a GPU** — even CPU-only CI runners like GitHub Actions can build GPU images.

```dockerfile
# syntax=docker/dockerfile:1.7
FROM python:3.12-slim-bookworm AS builder

# ... install dependencies ...

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

If you inject the HF token via BuildKit secret (`--mount=type=secret`), the token doesn't end up in image layers. The detailed workflow (GHCR push, GitHub Actions secret injection, etc.) is covered in the [Docker image](https://lemondouble.github.io/hayakoe/deploy/docker) page of the hayakoe docs.

If you go with a CPU-only image, PyTorch drops out and only ONNX Runtime stays, shrinking image size from gigabytes to a few hundred MB.

### Wrapping Up Part 4 — and the Series

Part 4 covered the four decisions a trained model went through to become a packaged library — API chaining, source abstraction, thread-safe singletons, Docker build patterns.

That brings the four-part hayakoe series to a close. It started lightly — "the TTS library I'd been using is starting to bug me, let me just rewrite it" — and before I knew it, I was covering model comparison, quantization, ONNX conversion, GPU acceleration, library design, and Docker builds.

#### Recap

- **[Intro](/en/p/ea78fbb9b78c38ac/)** — Why I decided to rewrite, and what came out of it
- **[Part 1](/en/p/4c2a6225a737f531/)** — Looking at TTS models again after 2 years and ending up back at VITS
- **[Part 2](/en/p/137243ede98030f1/)** — Cutting memory in half, making it 1.5× faster
- **[Part 3](/en/p/9a99dd816456c397/)** — Squeezing out the last 1%: torch.compile, batch inference, pause restoration, ARM64
- **Part 4 (this post)** — What I considered when packaging it as a library

#### Outputs

- **Repository**: [github.com/LemonDouble/hayakoe](https://github.com/LemonDouble/hayakoe)
- **Documentation**: [lemondouble.github.io/hayakoe](https://lemondouble.github.io/hayakoe/)
- **PyPI**: `pip install hayakoe`

#### And

The original motivation — "a TTS that's comfortable for me to use" — now works well. I use it daily for alarms and briefings, I no longer worry about an external download server going down, and I can keep up with Python version bumps without getting stuck.

What's left is Korean support. Currently hayakoe is based on the JP-Extra model, so it can't synthesize Korean speech. Adding it requires re-running pretraining with a Korean BERT (klue/roberta-large, etc.) + Korean G2P + public datasets. It takes time and a lot of GPU... but if I ever find the time, I'd like to give it a shot.

Thanks for reading this long series.
