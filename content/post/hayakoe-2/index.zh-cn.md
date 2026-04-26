---
title: "内存减半,速度提升1.5倍 - hayakoe Part 2"
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
    - 优化
---

_注：作者居住在韩国,部分内容包含韩国特有的背景。_

这是hayakoe系列的第二篇正文。前一篇文章可以在[hayakoe - 时隔2年再次研究各种TTS模型后又回归VITS的故事 - Part 1](/zh-cn/p/4c2a6225a737f531/)查看。

到[Part 1](/zh-cn/p/4c2a6225a737f531/)结束时,我已经拿到了训练好的模型,但正如系列[序言](/zh-cn/p/ea78fbb9b78c38ac/)中提到的,直接使用该模型有几个不便之处 — `openjtalk`从外部获取词典数据,服务器宕机时一起宕机的SPOF问题、由于该依赖被阻塞的Python版本升级、由于英语合成无法工作而通过翻译层绕过的问题,以及本文将讨论的内存和速度负担。

Part 2中,我将讲述如何减少其中的**内存和速度**。具体来说:

- **内存**:加载1个说话人时RAM **5,122 MB**
- **速度**:合成38.5秒文本耗时35.3秒 — 倍率 **1.09×**(CPU FP32基准)

要用作闹钟、简报用途,这两个数字都是负担。Part 2讲的是如何把这两个数字降到**2,346 MB · 3.6×**的故事。

大致流程如下。

1. **瓶颈测量** — 从时间花在哪里开始
2. **BERT量化** — 对内存有效,而非速度
3. **Synthesizer无法量化** — Flow层会崩
4. **ONNX Runtime图优化** — 找回Synthesizer速度
5. **89 MB内存事件** — 测量的陷阱

### 1. 瓶颈在哪里

最初想到的假设很简单。**看weight文件大小,BERT(DeBERTa v2 Large JP)约占整个模型的84%**,所以期待只量化BERT就能同时解决内存和速度。

但要验证那个假设,首先必须测量时间究竟花在哪里。我分别测量了BERT和Synthesizer(VITS)的推理时间(`time.perf_counter` 5次平均,PyTorch FP32 / CPU基准)。

| 文本 | BERT | Synthesizer | BERT占比 | Synth占比 |
|---|---|---|---|---|
| short (1.7s) | 0.489 s | 0.885 s | 36 % | **64 %** |
| medium (5.3s) | 0.602 s | 2.504 s | 19 % | **81 %** |
| long (7.8s) | 0.690 s | 3.714 s | 16 % | **84 %** |
| xlong (30s) | 1.074 s | 11.410 s | 9 % | **91 %** |

结果与预期完全相反。**CPU时间的64~91%被Synthesizer拿走**,文本越长,这个比重越大。

原因很简单。BERT对输入文本长度相对不敏感,而Synthesizer的时间会随生成的音频长度成正比增加。文本越长,需要合成的音频frame数越多,Synthesizer的Conv1d层会被反复调用相同的frame数。

也就是说分情况看:

- **短文本** — BERT虽然占36%左右的比重,但反正整个推理也只过1秒多,所以即便量化BERT,体感差异几乎没有。
- **长文本** — Synthesizer占到91%。这里就算把BERT做得再快,能省下的也只有9%,对实际加速没有大意义。

无论哪种,只攻BERT都难以做出**可感知的速度提升**。

> "没有测量的优化依赖直觉,而直觉常常是错的"——这是再次铭记这句格言的时刻。

### 2. BERT量化 — 内存而非速度

虽然瓶颈是Synthesizer这点已经明确,但这并不意味着BERT量化没有意义。**从内存而非速度的角度来看**。

BERT量化使用PyTorch的`torch.quantization.quantize_dynamic`应用。这是把Linear层的权重压缩到INT8,在推理时点动态进行量化、反量化的方式。

```python
import torch
from torch.quantization import quantize_dynamic

quantized_bert = quantize_dynamic(
    bert_model,
    {torch.nn.Linear},
    dtype=torch.qint8,
)
```

比较结果:

| 配置 | 推理时间 | RAM |
|---|---|---|
| PyTorch BERT FP32 | 4.796 s | +1,698 MB |
| PyTorch BERT Q8 | 4.536 s | **+368 MB** (−78 %) |

速度只提升了约5%(正如预期),但**内存减少了78%**。在一台服务器上跑多个程序,或容器内存限制紧张的环境下,这个差异会带来相当大的意义。

#### Q4 vs Q8 — 能减到哪里

从这里再深入一步,我尝试了INT4(Q4)。如果能进一步减少内存就好。

| 配置 | BERT大小 | RAM (1说话人) |
|---|---|---|
| FP32 | 1,157 MB | 1,599 MB |
| Q8 | 497 MB | 1,079 MB (−33 %) |
| Q4 | 394 MB | 958 MB (−40 %) |

不过验证音质后发现,FP32和Q8直接听都难以一致地区分,但**Q4在大部分段落相似,在句尾能听到细微差异。**

我判断额外能获得的内存收益(Q8 → Q4约−7 %p)不足以为听感损失正名,所以**默认值采用Q8**。

### 3. Synthesizer为什么没能量化

接下来自然出现的问题是"那Synthesizer也量化不就行了?"。因为时间都花在那里。

直接说结论 — **Synthesizer量化最终没有应用。** 我尝试了两个方向,但都没有显著效果。

#### 1. FP16转换(PyTorch) — Flow层会崩

在PyTorch中把Synthesizer转成FP16后,Flow层中`rational_quadratic_spline`这个函数因精度不足而崩溃,以一定概率触发以下assertion。

```
AssertionError: discriminant < 0
```

这个函数是按一定规则将输入变换得到输出的**变换函数**。VITS推理时**反向(inverse pass)**调用这个变换,过程中使用**二次方程的求根公式**。

从原版SBV2 [`transforms.py`](https://github.com/litagin02/Style-Bert-VITS2/blob/66de777e06392c0f313600be03c43ef96658b244/style_bert_vits2/models/transforms.py#L167) 的inverse分支摘录如下。

```python
# 二次方程 ax² + bx + c = 0 的系数
a = (inputs - input_cumheights) * (
    input_derivatives + input_derivatives_plus_one - 2 * input_delta
) + input_heights * (input_delta - input_derivatives)
b = input_heights * input_derivatives - (inputs - input_cumheights) * (
    input_derivatives + input_derivatives_plus_one - 2 * input_delta
)
c = -input_delta * (inputs - input_cumheights)

discriminant = b.pow(2) - 4 * a * c
assert (discriminant >= 0).all()        # ← 在这里崩

root = (2 * c) / (-b - torch.sqrt(discriminant))
```

判别式`b² - 4ac`为负时没有实根,变换不被定义,所以代码用`assert`阻断那个可能性。数学上当输入和weight在正常范围内时始终保证`≥ 0`,但**浮点数下情况就不一样了。** 降到FP16后精度不足产生微小的舍入误差,结果`discriminant`变为负数的情况发生,以一定概率触发assertion。

#### 2. INT8 dynamic quantization(ONNX Runtime) — 没有可量化的对象

接下来用ONNX Runtime的dynamic quantization尝试。这边只把权重存为INT8,激活仍以FP32流动,所以至少spline内的算术不会崩。

但尝试后发现还有另一个问题。ONNX Runtime的dynamic quantization**只量化`MatMul`运算**,而Synthesizer以`Conv1d`为主,事实上几乎没有可量化的对象。

| 模型 | FP32 | Q8 | 变化 |
|---|---|---|---|
| BERT (DeBERTa 330M,以MatMul为主) | 1,159 MB | 544 MB | **−47 %** |
| Synthesizer (以Conv1d为主) | 239 MB | 239 MB | **0 %** |

实际把Synthesizer量化为ONNX Q8后,模型文件大小不变,推理速度也几乎没有变化。

另外Synthesizer本身只有63M左右——是BERT的1/5水平——所以无论怎么量化,能获得的内存收益都不像BERT那样大。

那么Synthesizer的速度怎么办?这个问题自然跟着出现,而答案就是ONNX。

### 4. ONNX Runtime图优化

ONNX Runtime在加载模型时自动应用**图层级优化**。即使没有量化,也会经过以下变换来加快推理。

- **Kernel fusion** — 把连续的多个运算合并为一个。例如`Conv → BatchNorm → Activation`三步合并成一个fused kernel后,把中间结果写入内存再读取的开销消失,内存带宽得到节省。
- **Constant folding** — 把无论输入如何始终产生相同值的运算在加载时点预先计算。推理时直接使用预计算好的值。
- **删除不必要的节点** — 找出并删除未使用、重复或无意义的运算节点。

此外,ONNX Runtime通过[intra-op parallelism](https://onnxruntime.ai/docs/performance/tune-performance/threading.html) 把单个运算分布到多个CPU核心。即使同时请求只有一个,也能利用整个CPU,对单说话人、实时推理场景有利。

#### 应用结果 — CPU倍率

倍率 = 音频长度 / 推理时间(值越大越快)。

| 配置 | short (1.7s) | medium (7.6s) | long (10.7s) | xlong (38.5s) |
|---|---|---|---|---|
| SBV2 PyTorch FP32 | 1.52× | 2.27× | 2.16× | 1.09× |
| SBV2 ONNX FP32 | 1.76× | 3.09× | 3.26× | 2.75× |
| **HayaKoe (Q8 BERT + FP32 ONNX)** | **2.50×** | **3.35×** | **3.33×** | **3.60×** |

以xlong文本(38.5秒)为基准,原版PyTorch的1.09×在HayaKoe中**升到了3.60×**。从勉强跟上实时的水平,到能以约3倍的速度处理同样的输入。

#### 内存 — 加上BERT Q8效果

| 配置 | RAM (1说话人) |
|---|---|
| SBV2 PyTorch FP32 | 5,122 MB |
| SBV2 ONNX FP32 | 2,967 MB |
| **HayaKoe (Q8 BERT + FP32 ONNX)** | **2,346 MB** (−54 %) |

仅ONNX转换就让RAM减少约42%(因为去掉了PyTorch overhead),再加上BERT Q8量化进一步削减内存,最终达到−54%。

### 5. 89 MB内存事件 — 测量的陷阱

到此为止的数字看起来都很合理,但**为了让这些数字变得可信**,我曾大走弯路一次。

最初写benchmark时很简单。在一个Python进程内依次加载PyTorch模型 → ONNX FP32 → ONNX Q8,在每个时点测量RAM并比较。

```python
# 伪代码
result["pytorch"]      = measure(load_pytorch_model)
result["onnx_fp32"]    = measure(load_onnx_fp32)
result["onnx_all_q8"]  = measure(load_onnx_q8)  # ← 这里出现89 MB
```

但看结果JSON时发现了奇怪的值。

```
onnx_all_q8 RAM: 89 MB
```

**89 MB。** 即使Q8模型再小,光BERT INT8 + Synthesizer FP32 weight加起来也应该接近1 GB。实际把同一个模型作为独立进程启动时约为1,757 MB,但单进程测量中却显示为89 MB。

#### 追查原因 — Python内存分配器

精确断定原因很难,但根据观察行为推测的假设是 — **可能是下一个模型直接复用了上一个模型抓住的内存区域** — 。

- 加载第一个模型(PyTorch ~2,700 MB) → 进程RSS上升到2.7 GB
- 用`del`释放第一个模型 → Python对象图中消失,但从OS角度看,进程似乎仍持有那个区域
- 加载第二个模型 → 推测没有向OS要新内存,而是复用了之前释放的区域
- `psutil`看到的RSS delta在加载第二个模型时几乎为0 → 只有少量的89 MB追加被记录

也就是说,测量本身是看到了"那个时点的RSS精确变化",但对我们想知道的"这个模型单独使用时占多少内存"这个问题,给出了错误的答案。

我也尝试了用`gc.collect()`或`del`强制释放,但没有出现有意义的差异,这也是支撑上述假设的旁证。

#### 解决 — 进程隔离

最终**把测量代码改成各config在独立subprocess中运行**的方式。PyTorch进程结束后OS回收其内存,接下来的ONNX进程从干净状态开始。

```python
# 伪代码
for config in ["pytorch", "onnx_fp32", "onnx_all_q8"]:
    result = subprocess.run(
        ["python", "measure_one.py", "--config", config],
        capture_output=True,
    )
    save_result(config, parse(result.stdout))
```

这样隔离后,`onnx_all_q8` RAM正常测量为约1,757 MB,这个结果最终成为上面展示的"2,346 MB / -54%"数值的依据。

### Part 2收尾

Part 2整理了原本5,122 MB · 1.09×的模型如何被打磨成2,346 MB · 3.6×。

总结一下 — **BERT通过Q8量化减少内存消耗**,**Synthesizer不做量化而是转换为ONNX只取图优化的效果**,最后花时间**让测量环境本身变得可信**。

这种程度对一般使用场景已经足够,但实际合成多句子时,会开始浮现另一些细节。比如GPU环境的额外加速、多句子BERT批处理推理,以及多句子合成中消失的自然pause之类。

下文请看[Part 3 — 直到最后1%:torch.compile、批处理推理、Pause恢复、ARM64](/zh-cn/p/9a99dd816456c397/)。
