---
title: "榨干最后1% — torch.compile、批量推理、Pause恢复、ARM64 - hayakoe Part 3"
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
    - 优化
    - PyTorch
    - 树莓派
    - Python
---

_注：作者居住在韩国，部分内容包含韩国特有的背景。_

这是hayakoe系列的第三篇正篇。上一篇文章可以在[内存减半，速度提升1.5倍 - hayakoe Part 2](/zh-cn/p/137243ede98030f1/)中查看。

在[Part 2](/zh-cn/p/137243ede98030f1/)中，我通过主干 — BERT Q8量化 + Synthesizer ONNX转换 — 在内存和速度方面取得了相当大的改善。不过，实际运营这个模型时，除了主干之外，开始注意到一些感觉还不够好的部分。这些都是想要把它们打磨成更好用的形式的工作。

Part 3将介绍这样修整过的四个主题。

1. **`torch.compile`** — GPU推理加速
2. **BERT GPU保持 + 批量推理** — 在多句合成中带来有意义的差异
3. **自然pause恢复** — 让多句分割合成时消失的标点后停顿重新生效
4. **ARM64构建** — 让它在树莓派4B上也能运行

### 1. `torch.compile` — GPU推理加速

`torch.compile`是PyTorch 2.0引入的功能，通过对模型图进行JIT编译（运行时动态编译）来获得额外的速度提升。它使用CUDA Graphs（将重复的GPU运算序列一次性打包重放的功能）来减少GPU调用开销，并在可能的地方将多个运算替换为合并到单个GPU内核的形式（fused内核）。

在hayakoe中，调用`prepare()`时如果device是CUDA则自动应用`torch.compile` — 从用户角度看，无需额外配置，只需开启GPU模式即可。

| 后端 | 短句 | 中句 | 长句 |
|---|---|---|---|
| PyTorch (CUDA) | 7.3× | 16.3× | 13.6× |
| `torch.compile` | 7.4× | 17.2× | **15.4×** |
| 提升幅度 | +1 % | +6 % | **+13 %** |

长句中提升幅度大的原因是，文本越长，Synthesizer调用的Conv内核数量越多，启动开销也相应累积。CUDA Graphs一次性吸收了这些开销。

不过，要获得这个效果需要**预热**。CUDA Graphs捕获并编译图需要时间，所以最初几次调用反而更慢。hayakoe通过`prepare(warmup=True)`选项运行约8次dummy推理，让用户在首次请求时不会看到编译成本。

### 2. BERT GPU保持 + 批量推理

我也一起处理了影响GPU路径效率的两个问题 — **不必要的GPU↔CPU往返**和**逐句单独调用BERT**。

#### 移除`.cpu()` — 保持GPU张量

原始SBV2的[BERT特征提取代码](https://github.com/litagin02/Style-Bert-VITS2/blob/66de777e06392c0f313600be03c43ef96658b244/style_bert_vits2/nlp/japanese/bert_feature.py#L61)中有这样的部分。

```python
# 原始SBV2 (style_bert_vits2/nlp/japanese/bert_feature.py)
res = torch.cat(res["hidden_states"][-3:-2], -1)[0].cpu()
```

在GPU上forward BERT之后，每次都对输出调用`.cpu()`将其降到CPU，但这个输出会立即传给Synthesizer（同样在GPU上），所以又得重新上传到GPU。结果每个句子都会发生**GPU → CPU → GPU往返**，这个往返本身成为一个小瓶颈。

我把原始代码改成下面这样，让BERT输出保持GPU张量原样流动。

```python
# hayakoe
res = torch.cat(res["hidden_states"][-3:-2], -1)[0].float()  # 保持GPU
```

之所以调用`.float()`而不是`.cpu()`是为了dtype统一（FP16 BERT和FP32 Synthesizer之间的转换），详细情况在[Part 2的BERT量化 / FP16转换部分](/zh-cn/p/137243ede98030f1/)中已经讨论过。

此外，我把BERT模型本身作为**全局单例**管理，即使有多个说话人，BERT也只会上GPU一次，所有说话人共享该实例。

#### 多句BERT批处理

为了prosody（韵律）稳定性，hayakoe将输入文本按标点分割并以句子为单位合成。因此，BERT被反复调用句子数量次的结构自然就跟着出现了。

GPU上每次调用运算都伴随固定成本（kernel launch overhead），所以句子越短，这个调用成本比实际运算时间还要大的低效就会累积。幸运的是BERT (DeBERTa) 是HuggingFace Transformer，原生支持batch输入，所以我**将所有句子打包成一个batch，BERT只forward一次**进行处理。

| 句子数 | 顺序 | 批量 | 速度提升 |
|---|---|---|---|
| 2 | 0.447 s | 0.364 s | **1.23×** |
| 4 | 0.812 s | 0.566 s | **1.43×** |
| 8 | 1.598 s | 1.121 s | **1.43×** |
| 16 | 2.972 s | 2.264 s | **1.31×** |

句子数越多，+23% ~ +43%的速度提升稳定地出现。内存差异在1.3 MB以内，实际上相同。

有趣的是，在CPU (ONNX) 上重复同样的实验几乎没有效果。测量后只显示+1% ~ −10%之间的噪声水平差异。

CPU上没有大的效果但损失也几乎没有，所以为了让GPU·CPU双方在相同代码路径上工作，我决定保持批处理开启。

### 3. 自然pause恢复

这是本Part中细节最多的部分。

#### 分割合成的副作用

如上节所述，hayakoe为了prosody（韵律）稳定性将多句文本按标点分开后，分别合成每个句子。整体合成长文本时倾向于音调被压扁或不稳定，所以为了稳定性和自然性引入了这种结构。

但是这种分割有一个副作用。**句子之间的自然停顿（pause）会消失**。

原始SBV2在通文合成中会在`.`、`!`、`?`等标点之后产生自然的停顿。但是按句子分割后，每个句子在标点处结束，下一个句子从头开始，所以标点后的停顿也一起消失了。在初始实现中，我尝试在句子之间插入固定80 ms的静音，但实际的自然停顿在0.3 ~ 0.6秒水平，所以80 ms太短了，结果造成"喘不过气"的不自然发话。

#### 原始SBV2是如何制造pause的

我追踪了原始SBV2在通文合成中产生自然停顿的原理。结论出乎意料地简单 — 是**Duration Predictor预测标点音素帧数的副作用**。

Duration Predictor原本是预测"每个音素发音多少帧"的模块。比如"安"是5帧、"宁"是4帧。但是`.`、`!`、`?`等标点也包含在音素序列中，Duration Predictor也会对这些标点音素预测帧数。预测的帧数就是该位置的停顿长度。

在分割合成中，由于在标点位置合成被切断，这个信息就被原样丢弃了。

#### 解决 — 单独运行Duration Predictor

问题和原因都明确后，解决方案也自然而然地出现了。

核心想法很简单。**让分割前的原文文本只通过TextEncoder + Duration Predictor，预先获取标点位置的帧数**。跳过Flow和Decoder（实际制作音频的部分）。

```
全文文本（分割前原文）
  │
  ├─ TextEncoder (G2P → 音素序列 → 嵌入)
  │
  ├─ Duration Predictor (按音素预测帧数)
  │     └─ 仅提取标点位置的帧数
  │
  └─ pause时间计算
        frames × hop_length / sample_rate = 秒
```

Synthesizer整体pass成本的大部分在Flow + Decoder中（参见[Part 2的瓶颈测量](/zh-cn/p/137243ede98030f1/)），所以只运行到Duration Predictor的成本相对于整体合成非常低。

在`hop_length = 512`、`sample_rate = 44100`的hayakoe默认设置下，1帧相当于约11.6 ms，所以如果标点 + 相邻blank token的总帧数是35：

```
35 × 512 / 44100 ≈ 0.41秒
```

这样可以在每个句子边界获得自然的pause时间。

#### ONNX支持 — 单独导出Duration Predictor

在PyTorch路径中可以单独调用模型内部模块，所以只挑Duration Predictor运行就行。但是`synthesizer.onnx`是把Synthesizer整体作为一个端到端图导出的形式，所以无法在中间只取出Duration Predictor输出来用。

为了解决这个问题，我额外导出了**只包含TextEncoder + Duration Predictor的独立ONNX模型**。

- 产物：`duration_predictor.onnx` (~30 MB, FP32)
- 在ONNX Runtime上运行
- 现有发布模型中如果没有这个文件，则静默回退到80 ms（向后兼容）

#### 结果

对同一文本自动预测的句子边界pause：

| 后端 | pause范围 |
|---|---|
| GPU (PyTorch) | 0.41 s ~ 0.55 s |
| CPU (ONNX) | 0.38 s ~ 0.57 s |

两个后端的差异落在SDP (Stochastic Duration Predictor) 的概率采样特性上发生的自然变动范围内。也就是说ONNX转换造成的质量损失实际上没有。

直接听多句样本就会发现差异相当明显。80 ms固定静音的Before句子几乎听起来贴在一起，而带有Duration Predictor预测pause的After则更接近人发话的呼吸节奏。

### 4. ARM64构建 — Raspberry Pi 4B

最后，我让hayakoe不仅在x86_64上，还**在aarch64 (ARM64) Linux上以相同代码运行**。

这之所以可能，是因为具备了两个条件。

- **ONNX Runtime**官方提供aarch64构建
- **`pyopenjtalk`**的自家fork（[`lemon-pyopenjtalk-prebuilt`](https://github.com/LemonDoubleHQ/lemon-pyopenjtalk-prebuilt)、[相关文章](https://blog.lemondouble.com/p/pyopenjtalk-prebuilt/)）构建aarch64 wheel，并将词典一起包含在包中

第二个条件也是在解决[序言](/zh-cn/p/ea78fbb9b78c38ac/)中提到的"`openjtalk`外部下载依赖"问题时自然得到的成果。

#### Raspberry Pi 4B实测

在Raspberry Pi 4B (Linux 6.8, aarch64, ONNX Runtime 1.23.2) 上测量的结果：

| 文本 | 推理时间 | 倍速 |
|---|---|---|
| 短 | 3.169 s | 0.3× |
| 中 | 13.042 s | 0.3× |
| 长 | 35.119 s | 0.3× |

约为实时的1/3水平，对话用途上不够。但我认为光是能在ARM板上运行就有意义 — 离线批量合成或在集群的ARM节点上以异步任务运行的场景都可以充分利用。

> 我预计Apple Silicon (macOS) 上也能运行，但因没有测试设备无法确认。

### Part 3结语

在Part 3中，除了大的优化主干之外，整理了**修剪草坪级别的细节四个**。

通过`torch.compile`在长句中再多赚+13%，通过BERT GPU保持·批量推理解决GPU路径的低效，通过单独分离Duration Predictor恢复多句分割合成的自然性，最后将动作范围扩展到Raspberry Pi 4B的工作。

至此，模型·推理方面的优化基本完成。在最后的Part中，我将介绍如何把它打包成其他人也能轻松拿来用的库 — API设计、multi-source、thread-safe单例服务、FastAPI / Docker模式。

继续阅读[Part 4 — 打包成库时考虑的事情](/zh-cn/p/b7791b4f6c0818da/)。
