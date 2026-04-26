---
title: "最後の1%まで — torch.compile、バッチ推論、Pause復元、ARM64 - hayakoe Part 3"
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
    - 最適化
    - PyTorch
    - Raspberry Pi
    - Python
---

_注：筆者は韓国在住のため、本文には韓国特有の文脈が含まれることがあります。_

hayakoeシリーズの3本目の本編です。前回の記事は[メモリは半分に、速度は1.5倍速くした話 - hayakoe Part 2](/ja/p/137243ede98030f1/)からご覧いただけます。

[Part 2](/ja/p/137243ede98030f1/)では大きな幹 — BERT Q8量子化 + Synthesizer ONNX変換 — でメモリと速度をかなり改善しました。ただ、このモデルを実際に運用してみると、大きな幹とは別に物足りなく感じられた部分が少しずつ見えてきました。もう少し使いやすい形に整えてみたいと思える作業でした。

Part 3では、そのように手を入れた4つのトピックを扱います。

1. **`torch.compile`** — GPU推論の高速化
2. **BERT GPU保持 + バッチ推論** — 多文合成で意味のある差
3. **自然なpause復元** — 多文を分割合成すると消える句読点後の間を再び生かす
4. **ARM64ビルド** — Raspberry Pi 4Bでも動くように

### 1. `torch.compile` — GPU推論の高速化

PyTorch 2.0から導入された`torch.compile`は、モデルグラフをJITコンパイル（実行時に動的にコンパイル）して追加の速度向上を得る機能です。CUDA Graphs（繰り返されるGPU演算シーケンスを一度にまとめて再実行する機能）でGPU呼び出しオーバーヘッドを減らし、可能な箇所では複数の演算を1つのGPUカーネルにまとめた形（fusedカーネル）に置き換えます。

hayakoeでは`prepare()`呼び出し時にdeviceがCUDAなら自動的に`torch.compile`を適用します — ユーザーから見れば、追加設定なしにGPUモードを有効にするだけです。

| バックエンド | 短文 | 中文 | 長文 |
|---|---|---|---|
| PyTorch (CUDA) | 7.3× | 16.3× | 13.6× |
| `torch.compile` | 7.4× | 17.2× | **15.4×** |
| 向上幅 | +1 % | +6 % | **+13 %** |

長文で向上幅が大きい理由は、テキストが長いほどSynthesizerが呼び出すConvカーネル数が増え、それだけ起動オーバーヘッドも累積するからです。CUDA Graphsがそのオーバーヘッドを一度に吸収してくれる形になります。

ただし、この効果を得るには**ウォームアップ**が必要です。CUDA Graphsがグラフをキャプチャしコンパイルするのに時間がかかるため、最初の数回の呼び出しはむしろ遅くなります。hayakoeは`prepare(warmup=True)`オプションで8回ほどダミー推論を回し、ユーザーが最初のリクエストでコンパイルコストを目にしないようにしています。

### 2. BERT GPU保持 + バッチ推論

GPU経路の効率を損なっていた2つ — **不要なGPU↔CPU往復**と**文ごとのBERT個別呼び出し** — も併せて手を入れました。

#### `.cpu()`削除 — GPUテンソル保持

オリジナルSBV2の[BERT特徴抽出コード](https://github.com/litagin02/Style-Bert-VITS2/blob/66de777e06392c0f313600be03c43ef96658b244/style_bert_vits2/nlp/japanese/bert_feature.py#L61)に次のような部分がありました。

```python
# オリジナルSBV2 (style_bert_vits2/nlp/japanese/bert_feature.py)
res = torch.cat(res["hidden_states"][-3:-2], -1)[0].cpu()
```

BERTをGPUでforwardした後、出力に`.cpu()`を呼び出して毎回CPUに降ろしていましたが、この出力はすぐにSynthesizer（同じくGPU）に渡されるので再びGPUに上げ直す必要があります。結果として文ごとに**GPU → CPU → GPUの往復**が発生し、この往復自体が小さなボトルネックになります。

オリジナルコードを次のように変えて、BERT出力がGPUテンソルのまま流れるようにしました。

```python
# hayakoe
res = torch.cat(res["hidden_states"][-3:-2], -1)[0].float()  # GPU保持
```

`.cpu()`の代わりに`.float()`だけを呼び出す理由はdtype統一（FP16 BERTとFP32 Synthesizer間のキャスト）のためで、詳しい事情は[Part 2のBERT量子化 / FP16キャスト部分](/ja/p/137243ede98030f1/)で扱いました。

加えてBERTモデル自体を**グローバルシングルトン**として管理し、話者が複数いてもBERTは一度だけGPUにロードされ、すべての話者がそのインスタンスを共有するようにしました。

#### 多文BERTのバッチ化

hayakoeはprosody（韻律）の安定性のため、入力テキストを句読点を基準に分割し、文単位で合成します。そのため、BERTが文の数だけ繰り返し呼び出される構造が自然と付いてきます。

GPUでは演算を呼び出すたびに一定の固定コスト（kernel launch overhead）がかかるため、文が短いと実際の演算時間よりこの呼び出しコストの方が大きい非効率が累積します。幸いBERT (DeBERTa) はHuggingFace Transformerなのでbatch入力を標準サポートしており、**すべての文を1つのバッチにまとめてBERTを1回だけforward**するように処理しました。

| 文数 | 順次 | バッチ | 速度向上 |
|---|---|---|---|
| 2 | 0.447 s | 0.364 s | **1.23×** |
| 4 | 0.812 s | 0.566 s | **1.43×** |
| 8 | 1.598 s | 1.121 s | **1.43×** |
| 16 | 2.972 s | 2.264 s | **1.31×** |

文数が増えるほど+23%~+43%の速度向上が安定して現れます。メモリ差は1.3 MB以内で事実上同じでした。

興味深いのは、同じ実験をCPU (ONNX) で繰り返すと効果がほぼないという点でした。測定すると+1%~−10%の間のノイズレベルの差しか出ません。

CPUでは大きな効果はないが損もほぼないので、GPU・CPUの両方で同じコード経路で動作するように、バッチ化はそのまま有効にしておく方針で整理しました。

### 3. 自然なpause復元

今回のPartで最もディテールが多い部分です。

#### 分割合成の副作用

上のセクションで述べたように、hayakoeはprosody（韻律）の安定性のため多文を句読点を基準に分けてから各文を別々に合成します。長いテキストをまるごと合成すると抑揚が潰れたり不安定になったりする傾向があり、安定性と自然さのために導入した構造です。

しかしこの分割には1つ副作用があります。**文と文の間の自然な間（pause）が消える**ということでした。

オリジナルSBV2は通文合成で`.`、`!`、`?`のような句読点の後に自然な間を作ってくれます。ところが文単位で分割すると各文が句読点で終わり次の文が最初から始まるので、句読点後の間も一緒に消えます。初期実装では文と文の間に固定80 msの無音を挿入してみましたが、実際の自然な間は0.3 ~ 0.6秒水準なので80 msは短すぎて、結果的に「息を継ぐ間もない」不自然な発話になってしまいました。

#### オリジナルSBV2はpauseをどうやって作ったのか

オリジナルSBV2が通文合成で自然な間を作る原理を追跡してみました。結論は意外と単純でした — **Duration Predictorが句読点音素のframe数を予測する副次効果**でした。

Duration Predictorは元々「各音素を何フレームの間発音するか」を予測するモジュールです。「あ」は5フレーム、「ん」は4フレームのように。ところが`.`、`!`、`?`のような句読点も音素シーケンスに含まれており、Duration Predictorはこの句読点音素にもframe数を予測します。予測されたframe数がそのままその位置での間の長さになるわけです。

分割合成では句読点位置で合成が切れるため、この情報がそのまま捨てられていました。

#### 解決 — Duration Predictorだけ別途実行

問題と原因が両方明らかになったので解決も自然と続きました。

核心アイデアはシンプルです。**分割前の原文テキストをTextEncoder + Duration Predictorまで通過させ、句読点位置のframe数を事前に得ておくこと**です。FlowとDecoder（実際のオーディオを作る部分）はスキップします。

```
全テキスト（分割前の原文）
  │
  ├─ TextEncoder (G2P → 音素列 → 埋め込み)
  │
  ├─ Duration Predictor (音素別frame数予測)
  │     └─ 句読点位置のframe数のみ抽出
  │
  └─ pause時間計算
        frames × hop_length / sample_rate = 秒
```

Synthesizer全体passのコストの大部分はFlow + Decoderにあるので（[Part 2のボトルネック測定](/ja/p/137243ede98030f1/)参照）、Duration Predictorまでだけ実行するコストは全体合成に比べて非常に低いです。

`hop_length = 512`、`sample_rate = 44100`のhayakoeデフォルト設定で1フレームは約11.6 msに相当するので、句読点 + 隣接blank tokenの合算frame数が35なら：

```
35 × 512 / 44100 ≈ 0.41秒
```

このように各文境界ごとに自然なpause時間を得ることができます。

#### ONNX対応 — Duration Predictorだけ別途export

PyTorch経路ではモデル内部のモジュールを個別に呼び出せるので、Duration Predictorだけ選んで実行すれば良いです。ところが`synthesizer.onnx`はSynthesizer全体を1つのend-to-endグラフとしてエクスポートした形なので、途中でDuration Predictor出力だけを取り出して使うことが不可能でした。

これを解決するため、**TextEncoder + Duration Predictorだけを含む別途ONNXモデル**を追加でエクスポートしました。

- 成果物：`duration_predictor.onnx` (~30 MB, FP32)
- ONNX Runtime上で実行
- 既存の配布モデルにこのファイルがなければ静かに80 msフォールバックで動作（後方互換）

#### 結果

同じテキストに対して自動予測された文境界pause：

| バックエンド | pause範囲 |
|---|---|
| GPU (PyTorch) | 0.41 s ~ 0.55 s |
| CPU (ONNX) | 0.38 s ~ 0.57 s |

2つのバックエンドの差はSDP (Stochastic Duration Predictor) の確率サンプリング特性上発生する自然変動幅の中に収まります。つまりONNX変換による品質損失は事実上ありません。

多文サンプルを直接聞いてみると差はかなり明確です。80 ms固定無音だったBeforeは文がほぼくっついて聞こえるのに対し、Duration Predictorが予測したpauseが入ったAfterは人の発話の呼吸の流れに近くなります。

### 4. ARM64ビルド — Raspberry Pi 4B

最後に、hayakoeはx86_64だけでなく**aarch64 (ARM64) Linuxでも同じコードで動作する**ようにしました。

これが可能なのは2つの条件が揃っているからです。

- **ONNX Runtime**がaarch64ビルドを公式提供
- **`pyopenjtalk`**の独自fork（[`lemon-pyopenjtalk-prebuilt`](https://github.com/LemonDoubleHQ/lemon-pyopenjtalk-prebuilt)、[関連記事](https://blog.lemondouble.com/p/pyopenjtalk-prebuilt/)）がaarch64 wheelをビルドし、辞書をパッケージに同梱

2つ目の条件は[序論](/ja/p/ea78fbb9b78c38ac/)で言及した「`openjtalk`外部ダウンロード依存」問題を解決する過程で自然に付いてきた成果物でもあります。

#### Raspberry Pi 4B実測

Raspberry Pi 4B (Linux 6.8, aarch64, ONNX Runtime 1.23.2) で測定した結果：

| テキスト | 推論時間 | 倍速 |
|---|---|---|
| 短い | 3.169 s | 0.3× |
| 中間 | 13.042 s | 0.3× |
| 長い | 35.119 s | 0.3× |

リアルタイムの約1/3水準なので対話用途には不足しています。ただARMボードで動くこと自体に意味があると思います — オフラインバッチ合成や、クラスタのARMノードで非同期タスクとして回すシナリオには十分活用可能です。

> Apple Silicon (macOS) でも動作すると予想していますが、テスト機材がなく確認はできていません。

### Part 3を終えて

Part 3では大きな最適化の幹のほかに**芝刈りレベルのディテール4つ**を整理しました。

`torch.compile`で長文で+13%をさらに稼ぎ、BERT GPU保持・バッチ推論でGPU経路の非効率を解消し、Duration Predictorを別途切り出して多文分割合成の自然さを回復し、最後にRaspberry Pi 4Bまで動作範囲を広げた作業でした。

これでモデル・推論面の最適化は事実上完了しました。最後のPartでは、他の人も簡単に持ち帰って使えるライブラリとしてどうパッケージングしたか — API設計、multi-source、thread-safeシングルトンサービング、FastAPI / Dockerパターン — を扱う予定です。

[Part 4 — ライブラリにパッケージングしながら考えたこと](/ja/p/b7791b4f6c0818da/)に続きます。
