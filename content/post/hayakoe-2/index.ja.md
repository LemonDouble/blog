---
title: "メモリは半分に、速度は1.5倍速く - hayakoe Part 2"
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
    - 量子化
    - Style-Bert-VITS2
    - Python
    - 最適化
---

_注：筆者は韓国在住のため、本文には韓国特有の文脈が含まれることがあります。_

hayakoeシリーズの2本目の本編です。前回の記事は[hayakoe - 2年ぶりにTTSモデルをいろいろ見てまたVITSに戻ってきた話 - Part 1](/ja/p/4c2a6225a737f531/)でご覧いただけます。

[Part 1](/ja/p/4c2a6225a737f531/) で学習済みモデルを手にしたものの、シリーズの[序論](/ja/p/ea78fbb9b78c38ac/) で触れたように、そのモデルをそのまま使うにはいくつかの不便がありました — `openjtalk` が外部から辞書データを取得し、そのサーバーが落ちると一緒に落ちるSPOFの問題、その依存性のために塞がれていたPythonバージョンアップ、英語合成ができないために翻訳レイヤーで迂回していた点、そして本記事で扱うメモリ・速度の負担までです。

Part 2では、この中で**メモリと速度**をどう削減したかを紹介します。具体的には：

- **メモリ**：1話者ロード時のRAM **5,122 MB**
- **速度**：38.5秒分のテキスト合成に35.3秒 — 速度倍率 **1.09×**（CPU FP32基準）

アラームやブリーフィング用途に使うには、どちらも負担の大きい数値でした。Part 2は、この2つの数値を**2,346 MB・3.6×**まで引き下げた話です。

大きな流れは次の通りです。

1. **ボトルネックの測定** — 時間がどこで使われているかから
2. **BERT量子化** — 速度ではなくメモリに効果
3. **Synthesizerは量子化不可** — Flowレイヤーが壊れる
4. **ONNX Runtimeグラフ最適化** — Synthesizerの速度を取り戻す
5. **メモリ89 MB事件** — 測定の落とし穴

### 1. どこがボトルネックか

最初に思いついた仮説は単純でした。**weightファイルのサイズを見ると、BERT（DeBERTa v2 Large JP）がモデル全体の約84%を占めている**ので、BERTだけ量子化すればメモリも速度もどちらも解決すると期待していました。

しかしその仮説を検証するには、時間が正確にどこで消費されているかから測定する必要がありました。BERTとSynthesizer（VITS）の推論時間を分離して測定しました（`time.perf_counter` 5回平均、PyTorch FP32 / CPU基準）。

| テキスト | BERT | Synthesizer | BERT比率 | Synth比率 |
|---|---|---|---|---|
| short (1.7s) | 0.489 s | 0.885 s | 36 % | **64 %** |
| medium (5.3s) | 0.602 s | 2.504 s | 19 % | **81 %** |
| long (7.8s) | 0.690 s | 3.714 s | 16 % | **84 %** |
| xlong (30s) | 1.074 s | 11.410 s | 9 % | **91 %** |

期待とは正反対の結果でした。**CPU時間の64～91%をSynthesizerが持っていき**、テキストが長くなるほどその比率はさらに大きくなりました。

理由は単純です。BERTは入力テキストの長さに比較的鈍感である一方、Synthesizerは生成するオーディオの長さに比例して時間が増えるからです。テキストが長いほど合成すべきオーディオframe数が多くなり、SynthesizerのConv1dレイヤーがそのframe数だけ繰り返し呼び出されます。

つまりケースを分けて見ると：

- **短いテキスト** — BERTが36%程度の比率を占めるとはいえ、どうせ推論全体が1秒余りなので、BERTを量子化しても体感の差はほとんどありません。
- **長いテキスト** — Synthesizerが91%まで持っていきます。ここでBERTだけ速くしたところで削れるのは9%だけなので、実質的な高速化には大きな意味がありません。

どちらにせよBERTだけを攻めても**体感可能な速度向上**を作るのは難しかったです。

> 「測定しない最適化は直感に頼り、直感はしばしば間違う」という格言を、もう一度心に刻み直す瞬間でした。

### 2. BERT量子化 — 速度ではなくメモリ

ボトルネックがSynthesizerであることは明確になりましたが、だからといってBERT量子化が無意味というわけではありませんでした。**速度ではなくメモリの面で**です。

BERT量子化はPyTorchの`torch.quantization.quantize_dynamic`で適用しました。LinearレイヤーのweightをINT8に圧縮し、推論時点で動的に量子化・逆量子化を行う方式です。

```python
import torch
from torch.quantization import quantize_dynamic

quantized_bert = quantize_dynamic(
    bert_model,
    {torch.nn.Linear},
    dtype=torch.qint8,
)
```

結果を比較すると：

| 構成 | 推論時間 | RAM |
|---|---|---|
| PyTorch BERT FP32 | 4.796 s | +1,698 MB |
| PyTorch BERT Q8 | 4.536 s | **+368 MB** (−78 %) |

速度は約5%向上にとどまりましたが（予想通り）、**メモリは78%削減されました**。1台のサーバーで複数のプログラムを動かしたり、コンテナのメモリ制限が厳しい環境では、この差はかなり意味のあるものになります。

#### Q4 vs Q8 — どこまで削れるか

ここからさらに一歩進んでINT4（Q4）まで試してみました。メモリをさらに削れるならその方がいいので。

| 構成 | BERTサイズ | RAM (1話者) |
|---|---|---|
| FP32 | 1,157 MB | 1,599 MB |
| Q8 | 497 MB | 1,079 MB (−33 %) |
| Q4 | 394 MB | 958 MB (−40 %) |

ただし音質を検証してみると、FP32とQ8は直接聴いても一貫して区別するのが難しいレベルでしたが、**Q4はほとんどの区間で類似していたものの、文末で微細な差異が聞き取れました。**

追加で得られるメモリのメリット（Q8 → Q4で約−7 %p）は、聴感の損失を正当化するには不十分だと判断し、**デフォルト値としてQ8**を採用しました。

### 3. Synthesizerはなぜ量子化できなかったか

次に自然と出てくる質問は「ではSynthesizerも量子化すればいいのでは？」でした。そこが時間を全部使っているのですから。

結論から言うと、**Synthesizer量子化は結局適用しませんでした。** 2つの方向性を試しましたが、どちらも目立った効果がなかったからです。

#### 1. FP16キャスト（PyTorch） — Flowレイヤーが壊れる

PyTorchでSynthesizerをFP16にキャストしてみたところ、Flowレイヤー内の`rational_quadratic_spline`という関数が精度不足で壊れ、一定確率で次のassertionが発生しました。

```
AssertionError: discriminant < 0
```

この関数は入力を一定の規則で変換して出力を作る**変換関数**です。VITS推論時にはこの変換を**逆方向（inverse pass）**で呼び出すのですが、その過程で**二次方程式の解の公式**を使います。

オリジナルのSBV2の[`transforms.py`](https://github.com/litagin02/Style-Bert-VITS2/blob/66de777e06392c0f313600be03c43ef96658b244/style_bert_vits2/models/transforms.py#L167)のinverseブランチから抜粋すると次の通りです。

```python
# 二次方程式 ax² + bx + c = 0 の係数
a = (inputs - input_cumheights) * (
    input_derivatives + input_derivatives_plus_one - 2 * input_delta
) + input_heights * (input_delta - input_derivatives)
b = input_heights * input_derivatives - (inputs - input_cumheights) * (
    input_derivatives + input_derivatives_plus_one - 2 * input_delta
)
c = -input_delta * (inputs - input_cumheights)

discriminant = b.pow(2) - 4 * a * c
assert (discriminant >= 0).all()        # ← ここで壊れる

root = (2 * c) / (-b - torch.sqrt(discriminant))
```

判別式`b² - 4ac`が負だと実数解がなく変換が定義されないため、コードは`assert`でその可能性をブロックしています。数学的には入力・weightが正常範囲内にあるとき常に`≥ 0`が保証されますが、**浮動小数点だと話が変わってきます。** FP16に落ちると精度が不足して微細な丸め誤差が発生し、その結果`discriminant`が負になるケースが発生して、一定確率でassertionが発火します。

#### 2. INT8 dynamic quantization（ONNX Runtime） — 量子化対象がない

次はONNX Runtimeのdynamic quantizationで試してみました。こちらはweightだけINT8で保存し活性化はFP32のまま流れる方式なので、少なくともspline内の算術が壊れることはありません。

ただし試してみると別の問題がありました。ONNX Runtimeのdynamic quantizationは**`MatMul`演算のみ量子化する**のですが、Synthesizerは`Conv1d`中心なので、事実上量子化対象自体がほとんどありません。

| モデル | FP32 | Q8 | 変化 |
|---|---|---|---|
| BERT (DeBERTa 330M、MatMul中心) | 1,159 MB | 544 MB | **−47 %** |
| Synthesizer (Conv1d中心) | 239 MB | 239 MB | **0 %** |

実際にSynthesizerをONNX Q8で量子化してみましたが、モデルファイルのサイズはそのままで、推論速度にもほとんど変化がありませんでした。

加えてSynthesizer自体が63M程度と小さく — BERTの1/5レベル — どう量子化しても得られるメモリのメリットがBERTほど大きくないという点もありました。

ではSynthesizerの速度はどうやって稼ぐのか？という疑問が自然と続き、その答えがONNXでした。

### 4. ONNX Runtimeグラフ最適化

ONNX Runtimeはモデルをロードする時に**グラフレベルの最適化**を自動で適用します。量子化なしでも次のような変換を経て推論を高速化してくれます。

- **Kernel fusion** — 連続した複数の演算を1つにまとめます。例えば`Conv → BatchNorm → Activation`の3段階が1つのfusedカーネルになると、中間結果をメモリに書いて再度読む費用がなくなり、メモリ帯域が節約されます。
- **Constant folding** — 入力に関係なく常に同じ値を返す演算をロード時点で事前計算しておきます。推論時には事前計算された値をそのまま使用します。
- **不要なノードの削除** — 使われていない、重複している、または無意味な演算ノードを見つけて削除します。

これに加え、ONNX Runtimeは[intra-op parallelism](https://onnxruntime.ai/docs/performance/tune-performance/threading.html)で単一の演算を複数のCPUコアに分散させます。同時リクエストが1つしかなくてもCPU全体を活用できるので、単一話者・リアルタイム推論シナリオに有利です。

#### 適用結果 — CPU倍率

倍率 = オーディオ長 / 推論時間（値が大きいほど速い）。

| 構成 | short (1.7s) | medium (7.6s) | long (10.7s) | xlong (38.5s) |
|---|---|---|---|---|
| SBV2 PyTorch FP32 | 1.52× | 2.27× | 2.16× | 1.09× |
| SBV2 ONNX FP32 | 1.76× | 3.09× | 3.26× | 2.75× |
| **HayaKoe (Q8 BERT + FP32 ONNX)** | **2.50×** | **3.35×** | **3.33×** | **3.60×** |

xlongテキスト（38.5秒分）基準で、オリジナルのPyTorchの1.09×がHayaKoeでは**3.60×まで上がりました**。リアルタイムをかろうじて追いかけるレベルから、同じ入力を約3倍速く処理できるようになったわけです。

#### メモリ — BERT Q8の効果まで合わせて

| 構成 | RAM (1話者) |
|---|---|
| SBV2 PyTorch FP32 | 5,122 MB |
| SBV2 ONNX FP32 | 2,967 MB |
| **HayaKoe (Q8 BERT + FP32 ONNX)** | **2,346 MB** (−54 %) |

ONNX変換だけでもRAMが約42%減少し（PyTorchのoverheadがなくなるので）、そこにBERT Q8量子化が追加でメモリを削って最終的に−54%になりました。

### 5. メモリ89 MB事件 — 測定の落とし穴

これまでの数値はどれももっともらしく見えますが、**この数値を信頼できるものにするまでに**一度大きく迷走しました。

最初にベンチマークを書いた時は単純でした。1つのPythonプロセス内でPyTorchモデル → ONNX FP32 → ONNX Q8の順に順次ロードし、各時点でRAMを測定して比較する方式でした。

```python
# 疑似コード
result["pytorch"]      = measure(load_pytorch_model)
result["onnx_fp32"]    = measure(load_onnx_fp32)
result["onnx_all_q8"]  = measure(load_onnx_q8)  # ← ここで89 MBが出る
```

ところが結果のJSONを見ていて、おかしな値を発見しました。

```
onnx_all_q8 RAM: 89 MB
```

**89 MB。** Q8モデルがいくら小さいとはいえ、BERT INT8 + Synthesizer FP32 weightだけ合わせてもほぼ1 GB近くになるはずです。実際に同じモデルを単独プロセスで起動してみると約1,757 MBが出るのに、単一プロセス測定では89 MBで記録されました。

#### 原因追跡 — Pythonメモリアロケータ

原因を正確に断定するのは難しかったですが、動作を見て推測した仮説は — **以前のモデルが掴んでいたメモリ領域を次のモデルがそのまま再利用したのではないか** — というものでした。

- 最初のモデル（PyTorch ~2,700 MB）ロード → プロセスRSSが2.7 GBまで上昇
- 最初のモデルを`del`で解放 → Pythonのオブジェクトグラフからは消えるが、OS視点ではプロセスがまだその領域を持っているように見える
- 2つ目のモデルロード → 新たにOSにメモリを要求せず、先ほど解放された領域を再利用したと推測される
- `psutil`で見たRSS deltaは2つ目のモデルロード時にほぼ0 → 小さな追加分の89 MBだけが捕捉される

つまり測定自体は「正確にその時点のRSS変化」を見てはいるのですが、私たちが知りたかった「このモデルを単独で使う時にどれだけメモリを食うのか」という質問には、誤った答えを与えていたわけです。

`gc.collect()`や`del`で強制解放も試してみましたが、有意な差は出てこず、これも上記仮説を裏付ける状況証拠でした。

#### 解決 — プロセス分離

結局**各configを独立したsubprocessで実行する**方式に測定コードを書き直しました。PyTorchプロセスが終わるとOSがそのメモリを回収し、次のONNXプロセスはきれいな状態で開始します。

```python
# 疑似コード
for config in ["pytorch", "onnx_fp32", "onnx_all_q8"]:
    result = subprocess.run(
        ["python", "measure_one.py", "--config", config],
        capture_output=True,
    )
    save_result(config, parse(result.stdout))
```

このように分離すると`onnx_all_q8` RAMが正常に約1,757 MBで測定され、その結果が結局上に示した「2,346 MB / -54%」の数値の根拠になりました。

### Part 2を終えて

Part 2では5,122 MB・1.09×だったモデルが、どのように2,346 MB・3.6×まで仕上がったかを整理しました。

要約すると — **BERTはQ8量子化でメモリ消費を削減し**、**Synthesizerは量子化の代わりにONNXに変換してグラフ最適化の効果だけを取り**、最後に**測定環境自体を信頼できるものにする**のに時間を使ったわけです。

これで一般的な使用シナリオには十分ですが、実際に複数文を合成していくと、また別のディテールが見え始めます。GPU環境での追加加速、複数文BERTバッチ推論、そして複数文合成で消える自然なpauseのようなものです。

[Part 3 — 最後の1%まで：torch.compile、バッチ推論、Pause復元、ARM64](/ja/p/9a99dd816456c397/) に続きます。
