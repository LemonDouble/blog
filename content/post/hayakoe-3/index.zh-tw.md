---
title: "榨乾最後1% — torch.compile、批次推論、Pause恢復、ARM64 - hayakoe Part 3"
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
    - 最佳化
    - PyTorch
    - 樹莓派
    - Python
---

_註：筆者居住於韓國，部分內容包含韓國特有的背景。_

這是hayakoe系列的第三篇正篇。上一篇文章可以在[記憶體減半，速度提升1.5倍 - hayakoe Part 2](/zh-tw/p/137243ede98030f1/)中查看。

在[Part 2](/zh-tw/p/137243ede98030f1/)中，我透過主幹 — BERT Q8量化 + Synthesizer ONNX轉換 — 在記憶體和速度方面取得了相當大的改善。不過，實際營運這個模型時，除了主幹之外，開始注意到一些感覺還不夠好的部分。這些都是想要把它們打磨成更好用的形式的工作。

Part 3將介紹這樣修整過的四個主題。

1. **`torch.compile`** — GPU推論加速
2. **BERT GPU保持 + 批次推論** — 在多句合成中帶來有意義的差異
3. **自然pause恢復** — 讓多句分割合成時消失的標點後停頓重新生效
4. **ARM64建置** — 讓它在樹莓派4B上也能執行

### 1. `torch.compile` — GPU推論加速

`torch.compile`是PyTorch 2.0引入的功能，透過對模型圖進行JIT編譯（執行時動態編譯）來獲得額外的速度提升。它使用CUDA Graphs（將重複的GPU運算序列一次性打包重放的功能）來減少GPU呼叫開銷，並在可能的地方將多個運算替換為合併到單個GPU核心的形式（fused核心）。

在hayakoe中，呼叫`prepare()`時如果device是CUDA則自動套用`torch.compile` — 從使用者角度看，無需額外設定，只需開啟GPU模式即可。

| 後端 | 短句 | 中句 | 長句 |
|---|---|---|---|
| PyTorch (CUDA) | 7.3× | 16.3× | 13.6× |
| `torch.compile` | 7.4× | 17.2× | **15.4×** |
| 提升幅度 | +1 % | +6 % | **+13 %** |

長句中提升幅度大的原因是，文字越長，Synthesizer呼叫的Conv核心數量越多，啟動開銷也相應累積。CUDA Graphs一次性吸收了這些開銷。

不過，要獲得這個效果需要**預熱**。CUDA Graphs擷取並編譯圖需要時間，所以最初幾次呼叫反而更慢。hayakoe透過`prepare(warmup=True)`選項執行約8次dummy推論，讓使用者在首次請求時不會看到編譯成本。

### 2. BERT GPU保持 + 批次推論

我也一起處理了影響GPU路徑效率的兩個問題 — **不必要的GPU↔CPU往返**和**逐句單獨呼叫BERT**。

#### 移除`.cpu()` — 保持GPU張量

原始SBV2的[BERT特徵擷取程式碼](https://github.com/litagin02/Style-Bert-VITS2/blob/66de777e06392c0f313600be03c43ef96658b244/style_bert_vits2/nlp/japanese/bert_feature.py#L61)中有這樣的部分。

```python
# 原始SBV2 (style_bert_vits2/nlp/japanese/bert_feature.py)
res = torch.cat(res["hidden_states"][-3:-2], -1)[0].cpu()
```

在GPU上forward BERT之後，每次都對輸出呼叫`.cpu()`將其降到CPU，但這個輸出會立即傳給Synthesizer（同樣在GPU上），所以又得重新上傳到GPU。結果每個句子都會發生**GPU → CPU → GPU往返**，這個往返本身成為一個小瓶頸。

我把原始程式碼改成下面這樣，讓BERT輸出保持GPU張量原樣流動。

```python
# hayakoe
res = torch.cat(res["hidden_states"][-3:-2], -1)[0].float()  # 保持GPU
```

之所以呼叫`.float()`而不是`.cpu()`是為了dtype統一（FP16 BERT和FP32 Synthesizer之間的轉換），詳細情況在[Part 2的BERT量化 / FP16轉換部分](/zh-tw/p/137243ede98030f1/)中已經討論過。

此外，我把BERT模型本身作為**全域單例**管理，即使有多個說話人，BERT也只會上GPU一次，所有說話人共用該實例。

#### 多句BERT批次化

為了prosody（韻律）穩定性，hayakoe將輸入文字按標點分割並以句子為單位合成。因此，BERT被反覆呼叫句子數量次的結構自然就跟著出現了。

GPU上每次呼叫運算都伴隨固定成本（kernel launch overhead），所以句子越短，這個呼叫成本比實際運算時間還要大的低效就會累積。幸運的是BERT (DeBERTa) 是HuggingFace Transformer，原生支援batch輸入，所以我**將所有句子打包成一個batch，BERT只forward一次**進行處理。

| 句子數 | 順序 | 批次 | 速度提升 |
|---|---|---|---|
| 2 | 0.447 s | 0.364 s | **1.23×** |
| 4 | 0.812 s | 0.566 s | **1.43×** |
| 8 | 1.598 s | 1.121 s | **1.43×** |
| 16 | 2.972 s | 2.264 s | **1.31×** |

句子數越多，+23% ~ +43%的速度提升穩定地出現。記憶體差異在1.3 MB以內，實際上相同。

有趣的是，在CPU (ONNX) 上重複同樣的實驗幾乎沒有效果。測量後只顯示+1% ~ −10%之間的雜訊水平差異。

CPU上沒有大的效果但損失也幾乎沒有，所以為了讓GPU·CPU雙方在相同程式碼路徑上運作，我決定保持批次化開啟。

### 3. 自然pause恢復

這是本Part中細節最多的部分。

#### 分割合成的副作用

如上節所述，hayakoe為了prosody（韻律）穩定性將多句文字按標點分開後，分別合成每個句子。整體合成長文字時傾向於音調被壓扁或不穩定，所以為了穩定性和自然性引入了這種結構。

但是這種分割有一個副作用。**句子之間的自然停頓（pause）會消失**。

原始SBV2在通文合成中會在`.`、`!`、`?`等標點之後產生自然的停頓。但是按句子分割後，每個句子在標點處結束，下一個句子從頭開始，所以標點後的停頓也一起消失了。在初始實作中，我嘗試在句子之間插入固定80 ms的靜音，但實際的自然停頓在0.3 ~ 0.6秒水平，所以80 ms太短了，結果造成「喘不過氣」的不自然發話。

#### 原始SBV2是如何製造pause的

我追蹤了原始SBV2在通文合成中產生自然停頓的原理。結論出乎意料地簡單 — 是**Duration Predictor預測標點音素幀數的副作用**。

Duration Predictor原本是預測「每個音素發音多少幀」的模組。比如「安」是5幀、「寧」是4幀。但是`.`、`!`、`?`等標點也包含在音素序列中，Duration Predictor也會對這些標點音素預測幀數。預測的幀數就是該位置的停頓長度。

在分割合成中，由於在標點位置合成被切斷，這個資訊就被原樣丟棄了。

#### 解決 — 單獨執行Duration Predictor

問題和原因都明確後，解決方案也自然而然地出現了。

核心想法很簡單。**讓分割前的原文文字只通過TextEncoder + Duration Predictor，預先取得標點位置的幀數**。跳過Flow和Decoder（實際製作音訊的部分）。

```
全文文字（分割前原文）
  │
  ├─ TextEncoder (G2P → 音素序列 → 嵌入)
  │
  ├─ Duration Predictor (按音素預測幀數)
  │     └─ 僅擷取標點位置的幀數
  │
  └─ pause時間計算
        frames × hop_length / sample_rate = 秒
```

Synthesizer整體pass成本的大部分在Flow + Decoder中（參見[Part 2的瓶頸測量](/zh-tw/p/137243ede98030f1/)），所以只執行到Duration Predictor的成本相對於整體合成非常低。

在`hop_length = 512`、`sample_rate = 44100`的hayakoe預設設定下，1幀相當於約11.6 ms，所以如果標點 + 相鄰blank token的總幀數是35：

```
35 × 512 / 44100 ≈ 0.41秒
```

這樣可以在每個句子邊界取得自然的pause時間。

#### ONNX支援 — 單獨匯出Duration Predictor

在PyTorch路徑中可以單獨呼叫模型內部模組，所以只挑Duration Predictor執行就行。但是`synthesizer.onnx`是把Synthesizer整體作為一個端到端圖匯出的形式，所以無法在中間只取出Duration Predictor輸出來用。

為了解決這個問題，我額外匯出了**只包含TextEncoder + Duration Predictor的獨立ONNX模型**。

- 產物：`duration_predictor.onnx` (~30 MB, FP32)
- 在ONNX Runtime上執行
- 現有發布模型中如果沒有這個檔案，則靜默回退到80 ms（向後相容）

#### 結果

對同一文字自動預測的句子邊界pause：

| 後端 | pause範圍 |
|---|---|
| GPU (PyTorch) | 0.41 s ~ 0.55 s |
| CPU (ONNX) | 0.38 s ~ 0.57 s |

兩個後端的差異落在SDP (Stochastic Duration Predictor) 的機率取樣特性上發生的自然變動範圍內。也就是說ONNX轉換造成的品質損失實際上沒有。

直接聽多句樣本就會發現差異相當明顯。80 ms固定靜音的Before句子幾乎聽起來貼在一起，而帶有Duration Predictor預測pause的After則更接近人發話的呼吸節奏。

### 4. ARM64建置 — Raspberry Pi 4B

最後，我讓hayakoe不僅在x86_64上，還**在aarch64 (ARM64) Linux上以相同程式碼執行**。

這之所以可能，是因為具備了兩個條件。

- **ONNX Runtime**官方提供aarch64建置
- **`pyopenjtalk`**的自家fork（[`lemon-pyopenjtalk-prebuilt`](https://github.com/LemonDoubleHQ/lemon-pyopenjtalk-prebuilt)、[相關文章](https://blog.lemondouble.com/p/pyopenjtalk-prebuilt/)）建置aarch64 wheel，並將辭典一起包含在套件中

第二個條件也是在解決[序言](/zh-tw/p/ea78fbb9b78c38ac/)中提到的「`openjtalk`外部下載相依性」問題時自然得到的成果。

#### Raspberry Pi 4B實測

在Raspberry Pi 4B (Linux 6.8, aarch64, ONNX Runtime 1.23.2) 上測量的結果：

| 文字 | 推論時間 | 倍速 |
|---|---|---|
| 短 | 3.169 s | 0.3× |
| 中 | 13.042 s | 0.3× |
| 長 | 35.119 s | 0.3× |

約為即時的1/3水平，對話用途上不夠。但我認為光是能在ARM板上執行就有意義 — 離線批次合成或在叢集的ARM節點上以非同步任務執行的情境都可以充分利用。

> 我預計Apple Silicon (macOS) 上也能運作，但因沒有測試設備無法確認。

### Part 3結語

在Part 3中，除了大的最佳化主幹之外，整理了**修剪草坪級別的細節四個**。

透過`torch.compile`在長句中再多賺+13%，透過BERT GPU保持·批次推論解決GPU路徑的低效，透過單獨分離Duration Predictor恢復多句分割合成的自然性，最後將動作範圍擴展到Raspberry Pi 4B的工作。

至此，模型·推論方面的最佳化基本完成。在最後的Part中，我將介紹如何把它打包成其他人也能輕鬆拿來用的函式庫 — API設計、multi-source、thread-safe單例服務、FastAPI / Docker模式。

繼續閱讀[Part 4 — 打包成函式庫時考慮的事情](/zh-tw/p/b7791b4f6c0818da/)。
