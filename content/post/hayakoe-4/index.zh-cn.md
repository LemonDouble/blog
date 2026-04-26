---
title: "打包成库时考虑的问题 — hayakoe Part 4"
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
    - 库
    - FastAPI
    - Docker
    - API 设计
---

_注：作者居住在韩国，部分内容包含韩国特有的背景。_

这是 hayakoe 系列的第四篇，也是最后一篇正篇。前一篇可在 [榨干最后 1% — torch.compile、批量推理、Pause 恢复、ARM64 - hayakoe Part 3](/zh-cn/p/9a99dd816456c397/) 查看。

如果说 [Part 1](/zh-cn/p/4c2a6225a737f531/) 到 [Part 3](/zh-cn/p/9a99dd816456c397/) 是关于模型和推理方面的工作，那么 Part 4 整理的是 **将这些成果打包成其他人 (以及未来的我) 都能轻松使用的过程中** 所考虑的问题。

大致分为四个部分。

1. **API 设计** — 类似 `from_pretrained` 的熟悉调用 + 分阶段链式调用
2. **Source 抽象** — 在一个实例中混用 `hf://` / `s3://` / `file://`
3. **Thread-safe 单例服务** — 在 FastAPI 同步和异步下都安全
4. **Docker 构建模式** — 构建时拉取所有模型，运行时不依赖网络

### 1. API 设计 — 熟悉的调用 + 分阶段链式调用

决定做库的时候最先确定的是"以什么 API 暴露出去"。最终采用的是把两种方式结合起来的形式 — 类似 HuggingFace `transformers` 的 `from_pretrained` 那样 **以名字加载模型的熟悉调用**，以及为了分离不同成本阶段的 **链式调用**。

先看熟悉的调用部分，`transformers` 是这样用的 — 只要传入模型名，就会自动下载 / 缓存 / 加载。

```python
from transformers import AutoModel

model = AutoModel.from_pretrained("bert-base-uncased")
```

hayakoe 也设计成只要传说话人名字就能产生同样的动作。在此基础上，为了分离不同成本的阶段加入了链式调用。

```python
from hayakoe import TTS

tts = TTS().load("jvnv-F1-jp").prepare()
tts.speakers["jvnv-F1-jp"].generate("こんにちは").save("output.wav")
```

把 `prepare()` 和实际合成 (`generate()`) 分开的理由很简单 — **重的成本要在服务器开始接收请求之前预先支付**，让实际请求只做合成并立即返回响应。如果把模型加载和 (在 CUDA 上的) `torch.compile` 成本都集中在 `prepare()` 里，从第一个请求开始就能稳定快速地处理。

#### 按说话人分离内存

库设计阶段我比较在意的一点是 **多说话人服务时的内存效率**。如果同样的 BERT 按每个说话人单独加载，服务 N 个说话人时 BERT 就会被加载 N 次，服务器内存会迅速被填满。

正如 [Part 2](/zh-cn/p/137243ede98030f1/) 中看到的，BERT (DeBERTa) 占整个模型约 84%，所以让说话人之间共享它直接关系到内存效率。

```
TTS (引擎 — 共享资源)
├── BERT (DeBERTa, ~329M)   ← 仅加载一次，所有说话人共享
│
├── speakers["jvnv-F1-jp"]  → Synthesizer + style vectors (~250MB)
├── speakers["jvnv-F2-jp"]  → ...
└── ...
```

由于这种结构，即使说话人增加，内存也不会线性暴涨。每增加一个说话人只增加约 250 ~ 300 MB (CPU RAM 大约 300 ~ 400 MB)，让在一台服务器上运行多个说话人的场景变得现实。

### 2. Source 抽象 — multi-source URI 路由

实际运营库时会发现，**说话人模型在哪里** 因情况而异。

- 官方默认说话人在 HuggingFace 公开仓库
- 自己训练的用户使用 private HF 仓库
- 如果已有 S3 等基础设施则使用 S3 (或者 R2 · MinIO 这类 S3 兼容存储)
- 开发中的模型 / 想另外备份则使用本地目录

如果对每个源都做下载代码分支处理，引擎本身会变臃肿，各自使用不同的缓存路径管理也会变难。所以我把所有源都 **隐藏在共同接口后面**，抽象成用户只需要改 URI 就能用同样的 API 工作。

```python
class Source(Protocol):
    def fetch(self, prefix: str) -> Path:
        """将 prefix/ 下所有文件下载到缓存并返回本地路径。"""
        ...

    def upload(self, prefix: str, local_dir: Path) -> None:
        """将 local_dir 内容上传到 prefix/ 下 (用于发布)。"""
        ...
```

按 URI scheme 自动路由。

| URI scheme | 实现 | 行为 |
|---|---|---|
| `hf://user/repo` | `HFSource` | 通过 `huggingface_hub.snapshot_download()` 下载 |
| `s3://bucket/prefix` | `S3Source` | 基于 `boto3`。通过 `AWS_ENDPOINT_URL_S3` 支持 S3 兼容端点 (R2 · MinIO 等) |
| `file:///abs/path` 或 `/abs/path` | `LocalSource` | 直接使用本地目录，不下载 |

在一个实例中混用多个源也可以。

```python
tts = (
    TTS(device="cuda")
    .load("jvnv-F1-jp")                                  # 官方 HF
    .load("my-voice", source="hf://me/private-voices")   # private HF
    .load("client-a", source="s3://tts-prod/voices")     # S3
    .load("dev-voice", source="file:///mnt/experiments") # 本地
    .prepare()
)
```

所有源都共享同一个缓存根目录 (`HAYAKOE_CACHE` 环境变量或默认的 `$CWD/hayakoe_cache`)，所以只需要管理一个目录。如果以后要支持新的存储，只要实现 `Source` 协议就行。

### 3. Thread-safe 单例服务 — FastAPI 同步·异步

这部分是在实际把库当作服务器运营过程中打磨最多的部分。

#### 不是单例服务就根本无法成立

最开始我想过简单的"每个请求都创建 `TTS()` 实例处理"这种模式行不行，但实际上不可能。

GPU 模式下 `prepare()` 会跑 `torch.compile`，第一次编译要花几十秒。每个请求都重新跑的话响应根本结束不了。即使是 CPU 模式，光模型加载就要几秒，所以一样。

所以 hayakoe 的推荐模式是 **在应用生命周期内保持一个 TTS 实例**。如果是 FastAPI，就在 lifespan 里构建一次然后挂到 `app.state` 上。

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    tts = TTS(device="cuda")
    for name in SPEAKERS:
        tts.load(name)
    tts.prepare(warmup=True)   # 包含 torch.compile
    app.state.tts = tts
    yield
```

`warmup=True` 会预先跑约 8 次假推理，让第一次实际请求不用承担编译成本。在服务环境里几乎总是开着的选项。

#### 并发请求安全 — per-speaker `threading.Lock`

服务器可能同时收到多个请求。同一说话人的并发合成请求需要共享 GPU/CPU 资源，本来就要串行处理，让用户每次调用都加锁很麻烦。

所以 `Speaker` 对象内部持有 `threading.Lock`，对同一说话人的并发调用会自动串行化。

- **同一说话人并发调用** → 串行 (等锁)
- **不同说话人并发调用** → 并行 (每个说话人有独立锁)

如果预先 `load` 多个说话人，说话人之间的并行处理会自动进行。

#### 异步处理器使用 `agenerate` / `astream`

在 FastAPI async 处理器中直接调用同步 `generate()`，合成时间内事件循环会停止。所以也一并提供 async 包装器，内部通过 `asyncio.to_thread` 卸载到工作线程。

| 同步 | 异步 |
|---|---|
| `speaker.generate(text)` | `await speaker.agenerate(text)` |
| `speaker.stream(text)` | `async for chunk in speaker.astream(text)` |

需要注意一点，`astream` 在生成器存活期间一直持有 per-speaker lock。如果中途断开 `async for`，其他请求可能会等很久，所以与 FastAPI 的 `StreamingResponse` 一起使用比较安全 (客户端连接断开时会自动关闭生成器)。

### 4. Docker 构建模式 — 构建时把模型烤进去

最后，针对运营环境常见的两个需求，也整理了 Docker 构建模式。

- **离线环境支持** — 只要有一次烤好的 Docker 镜像，就要能在没有外部网络或外部依赖的情况下立即开始推理。如果运行时还要从 HuggingFace · S3 等下载模型，第一次请求会太慢，或者网络被屏蔽时部署本身就会崩溃。
- **避免凭证泄漏** — 不希望把 HF token 或 S3 key 放在运行时环境的情况

解法很简单。**构建时把模型全部下载到镜像里**，让运行时容器只从缓存即时加载。

为此 hayakoe 提供了一个单独的 `TTS.pre_download()` 方法，**只填充缓存而不实际初始化**。关键是它 **没有 GPU 也能工作** — 即使在 GitHub Actions 这类 CPU 专用 CI runner 上也能构建 GPU 用镜像。

```dockerfile
# syntax=docker/dockerfile:1.7
FROM python:3.12-slim-bookworm AS builder

# ... 安装依赖 ...

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

通过 BuildKit secret (`--mount=type=secret`) 注入 HF token 的话，token 不会留在镜像层中。详细的工作流 (GHCR push、GitHub Actions secret 注入等) 在 hayakoe docs 的 [Docker 镜像](https://lemondouble.github.io/hayakoe/deploy/docker) 页面里讲了。

如果做成 CPU 专用镜像，PyTorch 会被去掉只保留 ONNX Runtime，镜像大小可以从 GB 级缩到几百 MB。

### Part 4 收尾 — 以及整个系列收尾

Part 4 整理了训练好的模型被打包成库之前经过的四个决定 — API 链式调用、源抽象、thread-safe 单例、Docker 构建模式。

到这里 hayakoe 四部曲系列就结束了。一开始本来轻松开始的 — "用着的 TTS 库慢慢开始让我不爽，重写一下吧" — 回过神来已经在讲模型比较、量化、ONNX 转换、GPU 加速、库设计、Docker 构建了。

#### 系列回顾

- **[序言](/zh-cn/p/ea78fbb9b78c38ac/)** — 为什么决定重写，成果是什么
- **[Part 1](/zh-cn/p/4c2a6225a737f531/)** — 时隔 2 年再看各种 TTS 模型，结果又回到 VITS 的故事
- **[Part 2](/zh-cn/p/137243ede98030f1/)** — 内存减半、速度提升 1.5 倍的故事
- **[Part 3](/zh-cn/p/9a99dd816456c397/)** — 榨干最后 1%：torch.compile、批量推理、Pause 恢复、ARM64
- **Part 4 (本文)** — 打包成库时考虑的问题

#### 成果

- **Repository**: [github.com/LemonDouble/hayakoe](https://github.com/LemonDouble/hayakoe)
- **Documentation**: [lemondouble.github.io/hayakoe](https://lemondouble.github.io/hayakoe/)
- **PyPI**: `pip install hayakoe`

#### 然后

最初动机的"我用着舒服的 TTS"现在工作得很好。每天用于闹钟·简报，再也不用担心外部下载服务器挂掉，Python 版本升级也能畅通无阻地跟上。

剩下的课题就是韩语支持。目前 hayakoe 基于 JP-Extra 模型，所以无法合成韩语语音，需要用韩语 BERT (klue/roberta-large 等) + 韩语 G2P + 公开数据集重新做预训练。耗时间也需要很多 GPU…如果哪天有时间想试试。

感谢您读完这篇长文。
