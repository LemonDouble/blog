---
title: "ライブラリとしてパッケージングする際に考えたこと — hayakoe Part 4"
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
    - ライブラリ
    - FastAPI
    - Docker
    - API設計
---

_注：筆者は韓国在住のため、本文には韓国特有の文脈が含まれることがあります。_

hayakoe シリーズ 4 本目、最後の本編です。前回の記事は [最後の 1% まで — torch.compile、バッチ推論、Pause 復元、ARM64 - hayakoe Part 3](/ja/p/9a99dd816456c397/) でご覧いただけます。

[Part 1](/ja/p/4c2a6225a737f531/) から [Part 3](/ja/p/9a99dd816456c397/) まではモデル・推論側の作業でしたが、Part 4 は **その成果物を他の人 (そして将来の自分) が手軽に使えるようにパッケージングする過程** で考えたことをまとめます。

大きく 4 つに分けてみました。

1. **API 設計** — `from_pretrained` のような馴染みのある呼び出し + 段階的なチェイニング
2. **Source 抽象化** — `hf://` / `s3://` / `file://` を 1 つのインスタンスで混ぜて使う
3. **Thread-safe シングルトン サービング** — FastAPI 同期・非同期どちらでも安全に
4. **Docker ビルドパターン** — ビルド時にモデルをすべて取得しておき、ランタイムはネットワークなしで

### 1. API 設計 — 馴染みのある呼び出し + 段階的チェイニング

ライブラリを作ると決めた時に最初に決めたのは「どんな API として公開するか」でした。結論的には 2 つを合わせた形に — HuggingFace `transformers` の `from_pretrained` のような **名前でモデルをロードする馴染みのある呼び出し**、そしてコストの異なるステップを分離する **チェイニング**。

まず馴染みのある呼び出し側を見ると、`transformers` はこう使います — モデル名を渡すだけで自動でダウンロード / キャッシュ / ロードまでやってくれます。

```python
from transformers import AutoModel

model = AutoModel.from_pretrained("bert-base-uncased")
```

hayakoe も話者名を渡すだけで同じ動作になるようにしました。さらに、コストの異なるステップを分離するためにチェイニングを加えました。

```python
from hayakoe import TTS

tts = TTS().load("jvnv-F1-jp").prepare()
tts.speakers["jvnv-F1-jp"].generate("こんにちは").save("output.wav")
```

`prepare()` と実際の合成 (`generate()`) を別にした理由はシンプルです — **重いコストはサーバーがリクエストを受ける前に先に支払っておき**、実際のリクエストでは合成だけ行って即座にレスポンスとして返すためです。モデルロードと (CUDA なら) `torch.compile` のコストまで `prepare()` にまとめておけば、最初のリクエストから安定して速く処理できます。

#### 話者ごとのメモリ分離

ライブラリ設計段階で気を配った部分の 1 つが **多話者サービング時のメモリ効率** でした。同じ BERT を話者ごとに別々にロードすると N 名サービングする時に BERT が N 回乗るので、サーバーメモリがすぐに埋まってしまいます。

[Part 2](/ja/p/137243ede98030f1/) で見たように BERT (DeBERTa) がモデル全体の約 84% を占めるので、これを話者間で共有するのはメモリ効率に直結します。

```
TTS (エンジン — 共有リソース)
├── BERT (DeBERTa, ~329M)   ← 1 回だけロード、すべての話者で共有
│
├── speakers["jvnv-F1-jp"]  → Synthesizer + style vectors (~250MB)
├── speakers["jvnv-F2-jp"]  → ...
└── ...
```

この構造のおかげで話者が増えてもメモリが線形に膨張しません。話者 1 名追加されるごとに増えるのは約 250 ~ 300 MB (CPU RAM 基準で約 300 ~ 400 MB) 程度で、1 サーバーで複数の話者を回すシナリオが現実的になります。

### 2. Source 抽象化 — multi-source URI ルーティング

ライブラリを運用していると話者モデルが **どこにあるか** が状況ごとに変わります。

- 公式のデフォルト話者は HuggingFace 公開リポジトリ
- 自分で学習したユーザーは private HF リポジトリ
- S3 などのインフラがすでにある場合は S3 (または R2 · MinIO のような S3 互換ストレージ)
- 開発中のモデル / 別途バックアップを取りたい場合はローカルディレクトリ

ソースごとにダウンロードコードを分岐処理するとエンジン本体が肥大化し、それぞれ異なるキャッシュパスを使うことになって管理が難しくなります。なので、すべてのソースを **共通インターフェースの裏に隠して**、ユーザーは URI だけ変えれば同じ API で動作するように抽象化しました。

```python
class Source(Protocol):
    def fetch(self, prefix: str) -> Path:
        """prefix/ 以下のすべてのファイルをキャッシュに取得し、ローカルパスを返す。"""
        ...

    def upload(self, prefix: str, local_dir: Path) -> None:
        """local_dir の内容を prefix/ 以下にアップロード (配布用)。"""
        ...
```

URI スキームによって自動でルーティングされます。

| URI スキーム | 実装 | 動作 |
|---|---|---|
| `hf://user/repo` | `HFSource` | `huggingface_hub.snapshot_download()` でダウンロード |
| `s3://bucket/prefix` | `S3Source` | `boto3` ベース。`AWS_ENDPOINT_URL_S3` で S3 互換エンドポイント (R2 · MinIO など) をサポート |
| `file:///abs/path` または `/abs/path` | `LocalSource` | ローカルディレクトリをそのまま使用、ダウンロードなし |

1 つのインスタンスで複数のソースを混ぜて使っても大丈夫です。

```python
tts = (
    TTS(device="cuda")
    .load("jvnv-F1-jp")                                  # 公式 HF
    .load("my-voice", source="hf://me/private-voices")   # private HF
    .load("client-a", source="s3://tts-prod/voices")     # S3
    .load("dev-voice", source="file:///mnt/experiments") # ローカル
    .prepare()
)
```

キャッシュはすべてのソースが同じルート (`HAYAKOE_CACHE` 環境変数またはデフォルトの `$CWD/hayakoe_cache`) を共有するので、1 つのディレクトリだけ管理すれば済みます。新しいストレージをサポートする必要が出てきたら、`Source` プロトコルだけ実装すれば終わる構造です。

### 3. Thread-safe シングルトン サービング — FastAPI 同期・非同期

ここはライブラリを実際にサーバーとして運用しながら最も多く磨いた部分です。

#### シングルトンでないとサービス自体が成り立たない

最初は単純に「リクエストごとに `TTS()` インスタンスを作って処理」のようなパターンも可能か考えましたが、これは事実上不可能です。

GPU モードでは `prepare()` が `torch.compile` を回すので最初のコンパイルに数十秒かかります。リクエストごとにこれを回したらレスポンスが終わりません。CPU モードでもモデルロードだけで数秒かかるので同じです。

なので hayakoe の推奨パターンは **アプリの寿命中 TTS インスタンスを 1 つ維持する** ことです。FastAPI なら lifespan で 1 回ビルドして `app.state` に付ける形。

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    tts = TTS(device="cuda")
    for name in SPEAKERS:
        tts.load(name)
    tts.prepare(warmup=True)   # torch.compile まで先に
    app.state.tts = tts
    yield
```

`warmup=True` はダミー推論を 8 回程度先に回して、最初の実際のリクエストがコンパイルコストを背負わないようにします。サービング環境ではほぼ常にオンにするオプションです。

#### 同時リクエスト安全性 — per-speaker `threading.Lock`

サーバーでは同時に複数のリクエストが入ってくることがあります。同じ話者に対する同時合成リクエストは GPU/CPU リソースを共有しないといけないのでどうせ直列で処理されるべきですが、これを呼び出しごとにユーザーがロックを取るのは面倒です。

なので `Speaker` オブジェクトの中に `threading.Lock` を持たせて、同じ話者に対する同時呼び出しは自動でシリアライズされます。

- **同じ話者の同時呼び出し** → 直列 (ロック待ち)
- **異なる話者の同時呼び出し** → 並列 (各話者が別ロック)

複数の話者を事前に `load` しておけば、話者間の並列処理は自動で行われます。

#### 非同期ハンドラは `agenerate` / `astream`

FastAPI async ハンドラで同期 `generate()` をそのまま呼ぶと合成時間中イベントループが止まります。なので async ラッパーも一緒に提供し、内部的には `asyncio.to_thread` でワーカースレッドにオフロードします。

| 同期 | 非同期 |
|---|---|
| `speaker.generate(text)` | `await speaker.agenerate(text)` |
| `speaker.stream(text)` | `async for chunk in speaker.astream(text)` |

1 つ注意することは、`astream` はジェネレーターが生きている間 per-speaker lock を握っているという点です。`async for` を途中で切ると他のリクエストが長時間待たされる可能性があるので、FastAPI の `StreamingResponse` と一緒に使うのが安全です (クライアント接続が切れると自動でジェネレーターを閉じてくれます)。

### 4. Docker ビルドパターン — ビルド時にモデルまで焼き込む

最後に、運用環境でよく出会う 2 つの要件のために Docker ビルドパターンも整えました。

- **オフライン環境対応** — 一度焼いた Docker イメージさえあれば、外部ネットワークや外部依存なしで即座に推論を始められないといけません。ランタイムに HuggingFace · S3 などからモデルをダウンロードする段階が挟まると最初のリクエストが遅すぎたり、ネットワークが塞がれた時にデプロイ自体が壊れたりするからです。
- **クレデンシャル露出回避** — HF トークンや S3 キーをランタイム環境に置きたくない場合

解決策はシンプルです。**ビルド時にモデルをイメージの中にすべて取得しておき**、ランタイムコンテナはキャッシュから即座にロードするだけにする分離。

このために `TTS.pre_download()` という、**モデルをキャッシュにだけ満たして実際の初期化はしない** メソッドを別途提供します。重要なのはこれが **GPU なしでも動作する** という点 — GitHub Actions のような CPU 専用 CI ランナーでも GPU 用イメージをビルドできます。

```dockerfile
# syntax=docker/dockerfile:1.7
FROM python:3.12-slim-bookworm AS builder

# ... 依存関係インストール ...

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

BuildKit secret (`--mount=type=secret`) で HF トークンを注入すればトークンがイメージレイヤーに残りません。詳細なワークフロー (GHCR push、GitHub Actions secret 注入など) は hayakoe docs の [Docker イメージ](https://lemondouble.github.io/hayakoe/deploy/docker) ページで扱いました。

CPU 専用イメージにすると PyTorch が抜けて ONNX Runtime だけ入るので、イメージサイズが GB 単位から数百 MB まで減ります。

### Part 4 を終えて — そしてシリーズを終えて

Part 4 では学習されたモデルがライブラリとしてパッケージングされるまで経た 4 つの決定 — API チェイニング、ソース抽象化、thread-safe シングルトン、Docker ビルドパターン — をまとめました。

ここまでが hayakoe 4 部作シリーズです。最初は気軽に始めたのに — 「使っていた TTS ライブラリがそろそろ気になってきたから一度作り直してみよう」 — 気づいたらモデル比較、量子化、ONNX 変換、GPU アクセラレーション、ライブラリ設計、Docker ビルドまで扱っていました。

#### シリーズを振り返る

- **[序論](/ja/p/ea78fbb9b78c38ac/)** — なぜ作り直すことにしたか、成果物は何か
- **[Part 1](/ja/p/4c2a6225a737f531/)** — 2 年ぶりに TTS モデルをいろいろ見てまた VITS に落ち着いた話
- **[Part 2](/ja/p/137243ede98030f1/)** — メモリは半分に、速度は 1.5 倍速くした話
- **[Part 3](/ja/p/9a99dd816456c397/)** — 最後の 1% まで: torch.compile、バッチ推論、Pause 復元、ARM64
- **Part 4 (この記事)** — ライブラリとしてパッケージングする際に考えたこと

#### 成果物

- **Repository**: [github.com/LemonDouble/hayakoe](https://github.com/LemonDouble/hayakoe)
- **Documentation**: [lemondouble.github.io/hayakoe](https://lemondouble.github.io/hayakoe/)
- **PyPI**: `pip install hayakoe`

#### そして

最初の動機だった「自分にとって使いやすい TTS」は今や問題なく動いています。アラーム・ブリーフィング用途で毎日使っており、もう外部ダウンロードサーバーが落ちることを心配する必要もなく、Python のバージョンアップにも詰まることなくついていっています。

残った課題があるとすれば韓国語サポートです。現在 hayakoe は JP-Extra モデルベースなので韓国語音声は合成できないのですが、韓国語 BERT (klue/roberta-large など) + 韓国語 G2P + 公開データセットで事前学習をやり直す作業が必要です。時間がかかり GPU もたくさん必要なので…いつか時間ができたらやってみたいです。

長い記事をお読みいただきありがとうございました。
