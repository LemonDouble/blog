---
title: "hayakoe - 打造易用且快速的TTS函式庫 - 序論"
slug: ea78fbb9b78c38ac
date: 2026-04-26T11:31:45+09:00
image: "cover.png"
categories:
    - AI
    - TTS
tags:
    - AI
    - TTS
    - Style-Bert-VITS2
    - ONNX
    - 量化
    - Python
    - 函式庫
---

_註：筆者居住於韓國，部分內容包含韓國特有的背景。_

大約兩年前，我接觸到了一個叫做Bert-VITS2的日語TTS函式庫，並曾經寫過一篇整理學習方法的文章。

之後，我把訓練好的模型部署到家用叢集伺服器上日常使用。我對原始程式碼進行了Fork，然後只把語音合成所需的path抽取出來整理後使用。

但隨著時間推移，一些小麻煩漸漸累積起來。

- **將日語文本轉換為發音資訊的`openjtalk`函式庫，會持續從外部下載一些東西，但其實我也搞不太清楚它具體是怎麼運作的。** 雖然一直在用，但總有種「萬一那個下載伺服器掛了，我的伺服器豈不是也跟著掛」的不安感。
- **由於`openjtalk`，Python版本升級被卡住了。** 因為相依性相容性問題，長時間沒能升級，一直擱置著。
- **TTS不支援英文（變成無聲）。** 我用了在前面加一層翻譯層先轉成日語再合成的方式繞過去。雖然能跑，但小小的彆扭一直存在。
- 而決定性的一點是，**大部分原有函式庫都已經遷移到了一個叫fish-speech的新函式庫，而繼承自它的Style-Bert-VITS2 (SBV2) 也在8個月前停止了更新。**

> 反正都要維護，不如藉這個機會全部推倒重來，做成自己用著順手的樣子。

這就是這個專案的開始。而且既然要動手，那就順便深入研究模型程式碼，看看有沒有最佳化的餘地——這是第二個目的。

### 成果 — `hayakoe`

- **Repository**: [github.com/LemonDouble/hayakoe](https://github.com/LemonDouble/hayakoe)
- **Documentation**: [lemondouble.github.io/hayakoe](https://lemondouble.github.io/hayakoe/)
- **PyPI**: `pip install hayakoe`

最終大致變成了這樣使用的函式庫。

```python
from hayakoe import TTS

tts = TTS().load("jvnv-F1-jp").prepare()
tts.speakers["jvnv-F1-jp"].generate("こんにちは").save("output.wav")
```

特性總結一下：

- **去除`openjtalk`外部下載** — 字典一併打包，首次執行也無需依賴外部伺服器
- **支援最新Python** — 在最新Python 3.x環境直接安裝/執行
- **直接合成英文輸入 / 自訂字典註冊** — 內建22萬條目外來語字典自動轉換英文→片假名，未知專有名詞可直接註冊發音
- **CPU即時推論** — 比PyTorch快1.5×～3.6×，RAM減少約一半（5,122 MB → 2,346 MB，-54%）
- **無需torch** — CPU推論無需PyTorch即可執行，相依性更輕量
- **簡潔API** — `TTS().load(...).prepare()`一行即可啟動 / HuggingFace transformers風格
- **可插拔的來源** — 可在一個實例中混用`hf://`, `s3://`, `file://`
- **FastAPI伺服器整合支援** — 同時支援同步/非同步處理器
- **ARM64支援** — 可在Raspberry Pi 4B上執行，但速度較慢

### 「那你到底做了什麼讓它變快的？」

最初只是抱著「試試量化吧」這種輕鬆的想法開始的。

光看weight檔案大小，BERT在模型中佔的比重最大。所以我以為「只要量化這個就行了吧？」，但實際測量後發現並非如此。

- 文本越長，**Synthesizer佔用CPU時間的80～91%**，BERT量化能減少的時間還不到整體的5%。
- BERT INT8量化的實際價值不在速度，而在於**節省記憶體（1,698 MB → 368 MB，-78%）**。
- 「那量化Synthesizer不就行了」——但Synthesizer本身就無法量化，因為Flow層中的`rational_quadratic_spline`從FP16開始就會因數值不穩定而崩壞。

也就是說，如果不精確測量真正的瓶頸在哪裡，就會把時間花在錯誤的地方——這又是一次重新學到的教訓。

所以最終工作大致按這個流程進行。

1. **從模型開始重新選擇** — 既然過了兩年，反正都要重做，不如順便看看其他函式庫。我把據說韓語支援很好的GPT-SoVITS、自回歸類（Qwen-3-TTS等）作為候選進行了評估。由於是即時鬧鐘/簡報用途，延遲時間相當重要，最終落定在音質和速度平衡最好的Style-Bert-VITS2 (JP-Extra v2.7.0)。
   - **GPT-SoVITS** — 即時速度令人滿意，但推論語音的體感品質不如SBV2，淘汰。
   - **Qwen-TTS** — 運算量太大被淘汰。我原本的GPU是RTX 2070 Super，得知flash-attn只支援RTX 3000系以上的顯示卡後，就想「是不是沒裝flash-attn導致的？」，甚至專門買了RTX 3080，但生成時間還是太長，最終得出即時使用不切實際的結論。
2. **針對各元件套用不同的處方** — 基於測量結果，對BERT和Synthesizer分別套用了不同的最佳化。
   - **BERT** — 用INT8 dynamic quantization只減少記憶體。對速度影響不大，但要在一台伺服器上跑多個程式時，記憶體挺重要的。
   - **Synthesizer** — 轉換為ONNX並套用圖層級的最佳化。ONNX Runtime會自動套用kernel fusion（將多個運算合併為一個）、constant folding（在載入時預先計算常數運算）這類最佳化。

3. **完成瑣碎的細節** — 主線之外的細節也一併整理。
   - **`torch.compile`** — 在GPU推論上透過PyTorch JIT編譯進一步提速。
   - **BERT批次推論** — 多句合成時把BERT呼叫打包成一次，減少額外負擔。
   - **恢復自然停頓** — 原版SBV2在標點後會有自然的停頓，但分割合成多句時這些停頓就消失了。我把Duration Predictor單獨export成ONNX，讓它預測標點位置的停頓長度。
   - **ARM64建置** — 預先建置以便在Raspberry Pi 4B這類ARM裝置上也能執行。

4. **作為函式庫的設計** — 不只是讓它能動，而是要讓別人也能輕鬆拿來用，這部分我下了不少功夫。
   - **依說話者分離記憶體** — BERT、WavLM這類共享資源只載入一次，每個說話者只持有自己的權重。說話者增加記憶體也不會爆炸。
   - **multi-source支援** — 一個實例可以混用`hf://`, `s3://`, `file://`。官方模型用HuggingFace，已經在用S3就用S3，開發中的模型用本地，等等。
   - **Thread-safe單例服務** — 在FastAPI伺服器中，多個處理器共享一個TTS實例也是安全的。
   - **Docker建置模式** — 建置時把模型打入映像檔，執行時從快取即時載入。

### 接下來要寫的文章

這個系列計劃分為4部分。

1. **(Part 1) 時隔2年再次嘗試各種TTS模型，又一次落定到VITS的故事**
   嘗試GPT-SoVITS v4後為什麼換掉了，訓練資料集是怎麼蒐集的，10個checkpoint的聽感評估如何確定最佳step。
2. **(Part 2) 記憶體減半、速度提升1.5倍的故事**
   瓶頸測量方法論、量化實驗（為什麼Synthesizer無法量化）、ONNX轉換，以及記憶體被錯誤測量為89MB的那次事件。
3. **(Part 3) 直到最後1% — torch.compile、批次推論、Pause恢復、ARM64**
   GPU推論的精修，以及單獨把Duration Predictor抽出來export成ONNX的故事。
4. **(Part 4) 打包成函式庫時考慮的事情**
   API設計哲學、multi-source（`hf://` / `s3://` / `file://`）、thread-safe單例服務、FastAPI / Docker部署。

### 序論結語

最初寫訓練文章的時候，我還想著「訓練這一次就完事了」，但等模型真的拿到手，**怎麼讓別人也能輕鬆拿去用**才是比訓練本身大得多的工程。

在這個過程中我也踏入了平時不太接觸的量化、ONNX圖、ML服務領域，既然都做了，整理下來或許對走類似道路的人也有幫助，所以打算以系列形式留下記錄。
