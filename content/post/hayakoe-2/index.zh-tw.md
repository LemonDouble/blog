---
title: "記憶體減半,速度提升1.5倍 - hayakoe Part 2"
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
    - 量化
    - Style-Bert-VITS2
    - Python
    - 最佳化
---

_註:筆者居住於韓國,部分內容包含韓國特有的背景。_

這是hayakoe系列的第二篇正文。前一篇文章可以在[hayakoe - 時隔2年再次研究各種TTS模型後又回歸VITS的故事 - Part 1](/zh-tw/p/4c2a6225a737f531/)查看。

到[Part 1](/zh-tw/p/4c2a6225a737f531/)結束時,我已經拿到了訓練好的模型,但正如系列[序言](/zh-tw/p/ea78fbb9b78c38ac/)中提到的,直接使用該模型有幾個不便之處 — `openjtalk`從外部獲取詞典資料,伺服器當機時一起當機的SPOF問題、因該依賴被阻塞的Python版本升級、因英語合成無法運作而透過翻譯層繞過的問題,以及本文將討論的記憶體和速度負擔。

Part 2中,我將講述如何減少其中的**記憶體和速度**。具體來說:

- **記憶體**:載入1個說話人時RAM **5,122 MB**
- **速度**:合成38.5秒文本耗時35.3秒 — 倍率 **1.09×**(CPU FP32基準)

要用作鬧鐘、簡報用途,這兩個數字都是負擔。Part 2講的是如何把這兩個數字降到**2,346 MB · 3.6×**的故事。

大致流程如下。

1. **瓶頸測量** — 從時間花在哪裡開始
2. **BERT量化** — 對記憶體有效,而非速度
3. **Synthesizer無法量化** — Flow層會崩
4. **ONNX Runtime圖最佳化** — 找回Synthesizer速度
5. **89 MB記憶體事件** — 測量的陷阱

### 1. 瓶頸在哪裡

最初想到的假設很簡單。**看weight檔案大小,BERT(DeBERTa v2 Large JP)約佔整個模型的84%**,所以期待只量化BERT就能同時解決記憶體和速度。

但要驗證那個假設,首先必須測量時間究竟花在哪裡。我分別測量了BERT和Synthesizer(VITS)的推論時間(`time.perf_counter` 5次平均,PyTorch FP32 / CPU基準)。

| 文本 | BERT | Synthesizer | BERT佔比 | Synth佔比 |
|---|---|---|---|---|
| short (1.7s) | 0.489 s | 0.885 s | 36 % | **64 %** |
| medium (5.3s) | 0.602 s | 2.504 s | 19 % | **81 %** |
| long (7.8s) | 0.690 s | 3.714 s | 16 % | **84 %** |
| xlong (30s) | 1.074 s | 11.410 s | 9 % | **91 %** |

結果與預期完全相反。**CPU時間的64~91%被Synthesizer拿走**,文本越長,這個比重越大。

原因很簡單。BERT對輸入文本長度相對不敏感,而Synthesizer的時間會隨生成的音訊長度成正比增加。文本越長,需要合成的音訊frame數越多,Synthesizer的Conv1d層會被反覆呼叫相同的frame數。

也就是說分情況看:

- **短文本** — BERT雖然佔36%左右的比重,但反正整個推論也只過1秒多,所以即便量化BERT,體感差異幾乎沒有。
- **長文本** — Synthesizer佔到91%。這裡就算把BERT做得再快,能省下的也只有9%,對實際加速沒有大意義。

無論哪種,只攻BERT都難以做出**可感知的速度提升**。

> "沒有測量的最佳化依賴直覺,而直覺常常是錯的"——這是再次銘記這句格言的時刻。

### 2. BERT量化 — 記憶體而非速度

雖然瓶頸是Synthesizer這點已經明確,但這並不意味著BERT量化沒有意義。**從記憶體而非速度的角度來看**。

BERT量化使用PyTorch的`torch.quantization.quantize_dynamic`應用。這是把Linear層的權重壓縮到INT8,在推論時點動態進行量化、反量化的方式。

```python
import torch
from torch.quantization import quantize_dynamic

quantized_bert = quantize_dynamic(
    bert_model,
    {torch.nn.Linear},
    dtype=torch.qint8,
)
```

比較結果:

| 設定 | 推論時間 | RAM |
|---|---|---|
| PyTorch BERT FP32 | 4.796 s | +1,698 MB |
| PyTorch BERT Q8 | 4.536 s | **+368 MB** (−78 %) |

速度只提升了約5%(正如預期),但**記憶體減少了78%**。在一台伺服器上跑多個程式,或容器記憶體限制緊張的環境下,這個差異會帶來相當大的意義。

#### Q4 vs Q8 — 能減到哪裡

從這裡再深入一步,我嘗試了INT4(Q4)。如果能進一步減少記憶體就好。

| 設定 | BERT大小 | RAM (1說話人) |
|---|---|---|
| FP32 | 1,157 MB | 1,599 MB |
| Q8 | 497 MB | 1,079 MB (−33 %) |
| Q4 | 394 MB | 958 MB (−40 %) |

不過驗證音質後發現,FP32和Q8直接聽都難以一致地區分,但**Q4在大部分段落相似,在句尾能聽到細微差異。**

我判斷額外能獲得的記憶體收益(Q8 → Q4約−7 %p)不足以為聽感損失正名,所以**預設值採用Q8**。

### 3. Synthesizer為什麼沒能量化

接下來自然出現的問題是"那Synthesizer也量化不就行了?"。因為時間都花在那裡。

直接說結論 — **Synthesizer量化最終沒有應用。** 我嘗試了兩個方向,但都沒有顯著效果。

#### 1. FP16轉換(PyTorch) — Flow層會崩

在PyTorch中把Synthesizer轉成FP16後,Flow層中`rational_quadratic_spline`這個函式因精度不足而崩潰,以一定機率觸發以下assertion。

```
AssertionError: discriminant < 0
```

這個函式是按一定規則將輸入變換得到輸出的**變換函式**。VITS推論時**反向(inverse pass)**呼叫這個變換,過程中使用**二次方程式的求根公式**。

從原版SBV2 [`transforms.py`](https://github.com/litagin02/Style-Bert-VITS2/blob/66de777e06392c0f313600be03c43ef96658b244/style_bert_vits2/models/transforms.py#L167) 的inverse分支摘錄如下。

```python
# 二次方程式 ax² + bx + c = 0 的係數
a = (inputs - input_cumheights) * (
    input_derivatives + input_derivatives_plus_one - 2 * input_delta
) + input_heights * (input_delta - input_derivatives)
b = input_heights * input_derivatives - (inputs - input_cumheights) * (
    input_derivatives + input_derivatives_plus_one - 2 * input_delta
)
c = -input_delta * (inputs - input_cumheights)

discriminant = b.pow(2) - 4 * a * c
assert (discriminant >= 0).all()        # ← 在這裡崩

root = (2 * c) / (-b - torch.sqrt(discriminant))
```

判別式`b² - 4ac`為負時沒有實根,變換不被定義,所以程式碼用`assert`阻斷那個可能性。數學上當輸入和weight在正常範圍內時始終保證`≥ 0`,但**浮點數下情況就不一樣了。** 降到FP16後精度不足產生微小的捨入誤差,結果`discriminant`變為負數的情況發生,以一定機率觸發assertion。

#### 2. INT8 dynamic quantization(ONNX Runtime) — 沒有可量化的對象

接下來用ONNX Runtime的dynamic quantization嘗試。這邊只把權重存為INT8,啟用值仍以FP32流動,所以至少spline內的算術不會崩。

但嘗試後發現還有另一個問題。ONNX Runtime的dynamic quantization**只量化`MatMul`運算**,而Synthesizer以`Conv1d`為主,事實上幾乎沒有可量化的對象。

| 模型 | FP32 | Q8 | 變化 |
|---|---|---|---|
| BERT (DeBERTa 330M,以MatMul為主) | 1,159 MB | 544 MB | **−47 %** |
| Synthesizer (以Conv1d為主) | 239 MB | 239 MB | **0 %** |

實際把Synthesizer量化為ONNX Q8後,模型檔案大小不變,推論速度也幾乎沒有變化。

另外Synthesizer本身只有63M左右——是BERT的1/5水平——所以無論怎麼量化,能獲得的記憶體收益都不像BERT那樣大。

那麼Synthesizer的速度怎麼辦?這個問題自然跟著出現,而答案就是ONNX。

### 4. ONNX Runtime圖最佳化

ONNX Runtime在載入模型時自動應用**圖層級最佳化**。即使沒有量化,也會經過以下變換來加快推論。

- **Kernel fusion** — 把連續的多個運算合併為一個。例如`Conv → BatchNorm → Activation`三步合併成一個fused kernel後,把中間結果寫入記憶體再讀取的開銷消失,記憶體頻寬得到節省。
- **Constant folding** — 把無論輸入如何始終產生相同值的運算在載入時點預先計算。推論時直接使用預計算好的值。
- **刪除不必要的節點** — 找出並刪除未使用、重複或無意義的運算節點。

此外,ONNX Runtime透過[intra-op parallelism](https://onnxruntime.ai/docs/performance/tune-performance/threading.html) 把單個運算分布到多個CPU核心。即使同時請求只有一個,也能利用整個CPU,對單說話人、即時推論場景有利。

#### 應用結果 — CPU倍率

倍率 = 音訊長度 / 推論時間(值越大越快)。

| 設定 | short (1.7s) | medium (7.6s) | long (10.7s) | xlong (38.5s) |
|---|---|---|---|---|
| SBV2 PyTorch FP32 | 1.52× | 2.27× | 2.16× | 1.09× |
| SBV2 ONNX FP32 | 1.76× | 3.09× | 3.26× | 2.75× |
| **HayaKoe (Q8 BERT + FP32 ONNX)** | **2.50×** | **3.35×** | **3.33×** | **3.60×** |

以xlong文本(38.5秒)為基準,原版PyTorch的1.09×在HayaKoe中**升到了3.60×**。從勉強跟上即時的水平,到能以約3倍的速度處理同樣的輸入。

#### 記憶體 — 加上BERT Q8效果

| 設定 | RAM (1說話人) |
|---|---|
| SBV2 PyTorch FP32 | 5,122 MB |
| SBV2 ONNX FP32 | 2,967 MB |
| **HayaKoe (Q8 BERT + FP32 ONNX)** | **2,346 MB** (−54 %) |

僅ONNX轉換就讓RAM減少約42%(因為去掉了PyTorch overhead),再加上BERT Q8量化進一步削減記憶體,最終達到−54%。

### 5. 89 MB記憶體事件 — 測量的陷阱

到此為止的數字看起來都很合理,但**為了讓這些數字變得可信**,我曾大走彎路一次。

最初寫benchmark時很簡單。在一個Python行程內依序載入PyTorch模型 → ONNX FP32 → ONNX Q8,在每個時點測量RAM並比較。

```python
# 虛擬碼
result["pytorch"]      = measure(load_pytorch_model)
result["onnx_fp32"]    = measure(load_onnx_fp32)
result["onnx_all_q8"]  = measure(load_onnx_q8)  # ← 這裡出現89 MB
```

但看結果JSON時發現了奇怪的值。

```
onnx_all_q8 RAM: 89 MB
```

**89 MB。** 即使Q8模型再小,光BERT INT8 + Synthesizer FP32 weight加起來也應該接近1 GB。實際把同一個模型作為獨立行程啟動時約為1,757 MB,但單行程測量中卻顯示為89 MB。

#### 追查原因 — Python記憶體配置器

精確斷定原因很難,但根據觀察行為推測的假設是 — **可能是下一個模型直接重用了上一個模型抓住的記憶體區域** — 。

- 載入第一個模型(PyTorch ~2,700 MB) → 行程RSS上升到2.7 GB
- 用`del`釋放第一個模型 → Python物件圖中消失,但從OS角度看,行程似乎仍持有那個區域
- 載入第二個模型 → 推測沒有向OS要新記憶體,而是重用了之前釋放的區域
- `psutil`看到的RSS delta在載入第二個模型時幾乎為0 → 只有少量的89 MB追加被記錄

也就是說,測量本身是看到了"那個時點的RSS精確變化",但對我們想知道的"這個模型單獨使用時佔多少記憶體"這個問題,給出了錯誤的答案。

我也嘗試了用`gc.collect()`或`del`強制釋放,但沒有出現有意義的差異,這也是支撐上述假設的旁證。

#### 解決 — 行程隔離

最終**把測量程式碼改成各config在獨立subprocess中執行**的方式。PyTorch行程結束後OS回收其記憶體,接下來的ONNX行程從乾淨狀態開始。

```python
# 虛擬碼
for config in ["pytorch", "onnx_fp32", "onnx_all_q8"]:
    result = subprocess.run(
        ["python", "measure_one.py", "--config", config],
        capture_output=True,
    )
    save_result(config, parse(result.stdout))
```

這樣隔離後,`onnx_all_q8` RAM正常測量為約1,757 MB,這個結果最終成為上面展示的"2,346 MB / -54%"數值的依據。

### Part 2收尾

Part 2整理了原本5,122 MB · 1.09×的模型如何被打磨成2,346 MB · 3.6×。

總結一下 — **BERT透過Q8量化減少記憶體消耗**,**Synthesizer不做量化而是轉換為ONNX只取圖最佳化的效果**,最後花時間**讓測量環境本身變得可信**。

這種程度對一般使用場景已經足夠,但實際合成多句子時,會開始浮現另一些細節。比如GPU環境的額外加速、多句子BERT批次推論,以及多句子合成中消失的自然pause之類。

下文請看[Part 3 — 直到最後1%:torch.compile、批次推論、Pause還原、ARM64](/zh-tw/p/9a99dd816456c397/)。
