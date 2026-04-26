---
title: "打包成函式庫時考慮的問題 — hayakoe Part 4"
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
    - 函式庫
    - FastAPI
    - Docker
    - API 設計
---

_註：筆者居住於韓國，部分內容包含韓國特有的背景。_

這是 hayakoe 系列的第四篇，也是最後一篇正篇。前一篇可在 [榨乾最後 1% — torch.compile、批次推論、Pause 還原、ARM64 - hayakoe Part 3](/zh-tw/p/9a99dd816456c397/) 查看。

如果說 [Part 1](/zh-tw/p/4c2a6225a737f531/) 到 [Part 3](/zh-tw/p/9a99dd816456c397/) 是關於模型和推論方面的工作，那麼 Part 4 整理的是 **將那些成果打包成讓其他人 (以及未來的我) 都能輕鬆使用的過程中** 所考慮的問題。

大致分為四個部分。

1. **API 設計** — 類似 `from_pretrained` 的熟悉呼叫 + 分階段鏈式呼叫
2. **Source 抽象** — 在一個實例中混用 `hf://` / `s3://` / `file://`
3. **Thread-safe 單例服務** — 在 FastAPI 同步和非同步下都安全
4. **Docker 建置模式** — 建置時拉取所有模型，執行時不依賴網路

### 1. API 設計 — 熟悉的呼叫 + 分階段鏈式呼叫

決定做函式庫的時候最先確定的是「以什麼 API 公開出去」。最終採用的是把兩種方式結合起來的形式 — 類似 HuggingFace `transformers` 的 `from_pretrained` 那樣 **以名稱載入模型的熟悉呼叫**，以及為了分離不同成本階段的 **鏈式呼叫**。

先看熟悉的呼叫部分，`transformers` 是這樣用的 — 只要傳入模型名稱，就會自動下載 / 快取 / 載入。

```python
from transformers import AutoModel

model = AutoModel.from_pretrained("bert-base-uncased")
```

hayakoe 也設計成只要傳語者名稱就能產生同樣的動作。在此基礎上，為了分離不同成本的階段加入了鏈式呼叫。

```python
from hayakoe import TTS

tts = TTS().load("jvnv-F1-jp").prepare()
tts.speakers["jvnv-F1-jp"].generate("こんにちは").save("output.wav")
```

把 `prepare()` 和實際合成 (`generate()`) 分開的理由很簡單 — **重的成本要在伺服器開始接收請求之前預先支付**，讓實際請求只做合成並立即回傳回應。如果把模型載入和 (在 CUDA 上的) `torch.compile` 成本都集中在 `prepare()` 裡，從第一個請求開始就能穩定快速地處理。

#### 按語者分離記憶體

函式庫設計階段我比較在意的一點是 **多語者服務時的記憶體效率**。如果同樣的 BERT 按每個語者單獨載入，服務 N 個語者時 BERT 就會被載入 N 次，伺服器記憶體會迅速被填滿。

正如 [Part 2](/zh-tw/p/137243ede98030f1/) 中看到的，BERT (DeBERTa) 占整個模型約 84%，所以讓語者之間共享它直接關係到記憶體效率。

```
TTS (引擎 — 共享資源)
├── BERT (DeBERTa, ~329M)   ← 僅載入一次，所有語者共享
│
├── speakers["jvnv-F1-jp"]  → Synthesizer + style vectors (~250MB)
├── speakers["jvnv-F2-jp"]  → ...
└── ...
```

由於這種結構，即使語者增加，記憶體也不會線性暴增。每增加一個語者只增加約 250 ~ 300 MB (CPU RAM 大約 300 ~ 400 MB)，讓在一台伺服器上執行多個語者的場景變得實際可行。

### 2. Source 抽象 — multi-source URI 路由

實際營運函式庫時會發現，**語者模型在哪裡** 因情況而異。

- 官方預設語者在 HuggingFace 公開儲存庫
- 自己訓練的使用者使用 private HF 儲存庫
- 如果已有 S3 等基礎設施則使用 S3 (或者 R2 · MinIO 這類 S3 相容儲存)
- 開發中的模型 / 想另外備份則使用本地目錄

如果對每個來源都做下載程式分支處理，引擎本身會變臃腫，各自使用不同的快取路徑管理也會變難。所以我把所有來源都 **隱藏在共同介面後面**，抽象成使用者只需要改 URI 就能用同樣的 API 工作。

```python
class Source(Protocol):
    def fetch(self, prefix: str) -> Path:
        """將 prefix/ 下所有檔案下載到快取並回傳本地路徑。"""
        ...

    def upload(self, prefix: str, local_dir: Path) -> None:
        """將 local_dir 內容上傳到 prefix/ 下 (用於發布)。"""
        ...
```

按 URI scheme 自動路由。

| URI scheme | 實作 | 行為 |
|---|---|---|
| `hf://user/repo` | `HFSource` | 透過 `huggingface_hub.snapshot_download()` 下載 |
| `s3://bucket/prefix` | `S3Source` | 基於 `boto3`。透過 `AWS_ENDPOINT_URL_S3` 支援 S3 相容端點 (R2 · MinIO 等) |
| `file:///abs/path` 或 `/abs/path` | `LocalSource` | 直接使用本地目錄，不下載 |

在一個實例中混用多個來源也可以。

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

所有來源都共享同一個快取根目錄 (`HAYAKOE_CACHE` 環境變數或預設的 `$CWD/hayakoe_cache`)，所以只需要管理一個目錄。如果以後要支援新的儲存，只要實作 `Source` 協定就行。

### 3. Thread-safe 單例服務 — FastAPI 同步·非同步

這部分是在實際把函式庫當作伺服器營運過程中打磨最多的部分。

#### 不是單例服務就根本無法成立

最開始我想過簡單的「每個請求都建立 `TTS()` 實例處理」這種模式行不行，但實際上不可能。

GPU 模式下 `prepare()` 會跑 `torch.compile`，第一次編譯要花幾十秒。每個請求都重新跑的話回應根本結束不了。即使是 CPU 模式，光模型載入就要幾秒，所以一樣。

所以 hayakoe 的推薦模式是 **在應用程式生命週期內保持一個 TTS 實例**。如果是 FastAPI，就在 lifespan 裡建置一次然後掛到 `app.state` 上。

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

`warmup=True` 會預先跑約 8 次假推論，讓第一次實際請求不用承擔編譯成本。在服務環境裡幾乎總是開著的選項。

#### 並行請求安全 — per-speaker `threading.Lock`

伺服器可能同時收到多個請求。同一語者的並行合成請求需要共享 GPU/CPU 資源，本來就要序列化處理，讓使用者每次呼叫都加鎖很麻煩。

所以 `Speaker` 物件內部持有 `threading.Lock`，對同一語者的並行呼叫會自動序列化。

- **同一語者並行呼叫** → 序列 (等鎖)
- **不同語者並行呼叫** → 並行 (每個語者有獨立鎖)

如果預先 `load` 多個語者，語者之間的並行處理會自動進行。

#### 非同步處理器使用 `agenerate` / `astream`

在 FastAPI async 處理器中直接呼叫同步 `generate()`，合成時間內事件迴圈會停止。所以也一併提供 async 包裝器，內部透過 `asyncio.to_thread` 卸載到工作執行緒。

| 同步 | 非同步 |
|---|---|
| `speaker.generate(text)` | `await speaker.agenerate(text)` |
| `speaker.stream(text)` | `async for chunk in speaker.astream(text)` |

需要注意一點，`astream` 在產生器存活期間一直持有 per-speaker lock。如果中途中斷 `async for`，其他請求可能會等很久，所以與 FastAPI 的 `StreamingResponse` 一起使用比較安全 (用戶端連線中斷時會自動關閉產生器)。

### 4. Docker 建置模式 — 建置時把模型烤進去

最後，針對營運環境常見的兩個需求，也整理了 Docker 建置模式。

- **離線環境支援** — 只要有一次烤好的 Docker 映像檔，就要能在沒有外部網路或外部依賴的情況下立即開始推論。如果執行時還要從 HuggingFace · S3 等下載模型，第一次請求會太慢，或者網路被封鎖時部署本身就會崩潰。
- **避免憑證外洩** — 不希望把 HF token 或 S3 key 放在執行時環境的情況

解法很簡單。**建置時把模型全部下載到映像檔裡**，讓執行時容器只從快取即時載入。

為此 hayakoe 提供了一個獨立的 `TTS.pre_download()` 方法，**只填充快取而不實際初始化**。關鍵是它 **沒有 GPU 也能工作** — 即使在 GitHub Actions 這類 CPU 專用 CI runner 上也能建置 GPU 用映像檔。

```dockerfile
# syntax=docker/dockerfile:1.7
FROM python:3.12-slim-bookworm AS builder

# ... 安裝相依套件 ...

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

透過 BuildKit secret (`--mount=type=secret`) 注入 HF token 的話，token 不會留在映像檔層中。詳細的工作流程 (GHCR push、GitHub Actions secret 注入等) 在 hayakoe docs 的 [Docker 映像檔](https://lemondouble.github.io/hayakoe/deploy/docker) 頁面裡講了。

如果做成 CPU 專用映像檔，PyTorch 會被去掉只保留 ONNX Runtime，映像檔大小可以從 GB 級縮到幾百 MB。

### Part 4 收尾 — 以及整個系列收尾

Part 4 整理了訓練好的模型被打包成函式庫之前經過的四個決定 — API 鏈式呼叫、來源抽象、thread-safe 單例、Docker 建置模式。

到這裡 hayakoe 四部曲系列就結束了。一開始本來輕鬆開始的 — 「用著的 TTS 函式庫慢慢開始讓我不爽，重寫一下吧」 — 回過神來已經在講模型比較、量化、ONNX 轉換、GPU 加速、函式庫設計、Docker 建置了。

#### 系列回顧

- **[序言](/zh-tw/p/ea78fbb9b78c38ac/)** — 為什麼決定重寫，成果是什麼
- **[Part 1](/zh-tw/p/4c2a6225a737f531/)** — 時隔 2 年再看各種 TTS 模型，結果又回到 VITS 的故事
- **[Part 2](/zh-tw/p/137243ede98030f1/)** — 記憶體減半、速度提升 1.5 倍的故事
- **[Part 3](/zh-tw/p/9a99dd816456c397/)** — 榨乾最後 1%：torch.compile、批次推論、Pause 還原、ARM64
- **Part 4 (本文)** — 打包成函式庫時考慮的問題

#### 成果

- **Repository**: [github.com/LemonDouble/hayakoe](https://github.com/LemonDouble/hayakoe)
- **Documentation**: [lemondouble.github.io/hayakoe](https://lemondouble.github.io/hayakoe/)
- **PyPI**: `pip install hayakoe`

#### 然後

最初動機的「我用著舒服的 TTS」現在運作得很好。每天用於鬧鐘·簡報，再也不用擔心外部下載伺服器掛掉，Python 版本升級也能順暢地跟上。

剩下的課題就是韓語支援。目前 hayakoe 基於 JP-Extra 模型，所以無法合成韓語語音，需要用韓語 BERT (klue/roberta-large 等) + 韓語 G2P + 公開資料集重新做預訓練。耗時間也需要很多 GPU…如果哪天有時間想試試。

感謝您讀完這篇長文。
