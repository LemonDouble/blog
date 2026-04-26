---
title: "Cutting Memory in Half and Speeding Up by 1.5x - hayakoe Part 2"
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
    - Quantization
    - Style-Bert-VITS2
    - Python
    - Optimization
---

_Note: I'm based in Korea, so some context here is Korea-specific._

This is the second main installment of the hayakoe series. You can read the previous post at [hayakoe - After 2 Years, Trying Various TTS Models Again and Settling Back to VITS - Part 1](/en/p/4c2a6225a737f531/).

By the end of [Part 1](/en/p/4c2a6225a737f531/), I had a trained model in hand, but as I noted in the [introduction](/en/p/ea78fbb9b78c38ac/) of the series, using that model as-is came with several inconveniences — the SPOF issue where `openjtalk` fetches dictionary data from an external server and dies along with it, the Python version upgrade blocked by that dependency, the fact that English synthesis didn't work so I was routing through a translation layer, and the memory and speed burden I'll cover in this post.

In Part 2, I'll walk through how I reduced **memory and speed** among these. Specifically:

- **Memory**: 5,122 MB of RAM when loading 1 speaker
- **Speed**: 35.3 seconds to synthesize 38.5 seconds of text — speedup **1.09×** (CPU FP32 baseline)

Both numbers were a burden if I wanted to use this for alarms and briefings. Part 2 is the story of how I pulled those two numbers down to **2,346 MB · 3.6×**.

The overall flow goes like this:

1. **Bottleneck measurement** — figuring out where the time is spent first
2. **BERT quantization** — effective for memory, not speed
3. **Synthesizer can't be quantized** — Flow layer breaks
4. **ONNX Runtime graph optimization** — recovering Synthesizer speed
5. **The 89 MB memory incident** — the pitfalls of measurement

### 1. Where Is the Bottleneck?

The first hypothesis I came up with was simple. **Looking at weight file sizes, BERT (DeBERTa v2 Large JP) takes up about 84% of the entire model**, so I expected that quantizing only BERT would solve both memory and speed.

But to verify that hypothesis, I had to first measure exactly where the time was being spent. I separated and measured the inference times of BERT and Synthesizer (VITS) (`time.perf_counter`, average of 5 runs, PyTorch FP32 / CPU baseline).

| Text | BERT | Synthesizer | BERT % | Synth % |
|---|---|---|---|---|
| short (1.7s) | 0.489 s | 0.885 s | 36 % | **64 %** |
| medium (5.3s) | 0.602 s | 2.504 s | 19 % | **81 %** |
| long (7.8s) | 0.690 s | 3.714 s | 16 % | **84 %** |
| xlong (30s) | 1.074 s | 11.410 s | 9 % | **91 %** |

The result was completely opposite to my expectation. **The Synthesizer takes 64 ~ 91 % of the CPU time**, and the longer the text, the bigger that share gets.

The reason is simple. BERT is relatively insensitive to input text length, while the Synthesizer's time grows in proportion to the audio length to be generated. The longer the text, the more audio frames need to be synthesized, and the Synthesizer's Conv1d layers get called repeatedly that many times.

So broken down by case:

- **Short text** — BERT does take about 36 %, but the entire inference is just over 1 second anyway, so quantizing BERT barely changes the perceived speed.
- **Long text** — Synthesizer takes up to 91 %. Even if I made BERT lightning fast, all I could shave off is 9 %, which doesn't really matter for actual acceleration.

Either way, just tackling BERT wasn't going to produce a **perceivable speed improvement**.

> "Optimization without measurement relies on intuition, and intuition is often wrong" — this was a moment where I etched that adage into my mind once again.

### 2. BERT Quantization — Memory, Not Speed

It was clear the bottleneck was the Synthesizer, but that didn't mean BERT quantization was meaningless. **From a memory standpoint, not speed.**

I applied BERT quantization using PyTorch's `torch.quantization.quantize_dynamic`. It compresses Linear layer weights to INT8 and dynamically quantizes/dequantizes at inference time.

```python
import torch
from torch.quantization import quantize_dynamic

quantized_bert = quantize_dynamic(
    bert_model,
    {torch.nn.Linear},
    dtype=torch.qint8,
)
```

Comparing the results:

| Config | Inference Time | RAM |
|---|---|---|
| PyTorch BERT FP32 | 4.796 s | +1,698 MB |
| PyTorch BERT Q8 | 4.536 s | **+368 MB** (−78 %) |

Speed improved by only about 5 % (just as expected), but **memory dropped by 78 %**. In environments where multiple programs run on a single server or container memory limits are tight, this difference becomes quite meaningful.

#### Q4 vs Q8 — How Far Can We Go?

I went one step further and tried INT4 (Q4). It would be nice if I could reduce memory even more.

| Config | BERT Size | RAM (1 speaker) |
|---|---|---|
| FP32 | 1,157 MB | 1,599 MB |
| Q8 | 497 MB | 1,079 MB (−33 %) |
| Q4 | 394 MB | 958 MB (−40 %) |

However, when I verified audio quality, FP32 and Q8 were hard to consistently distinguish on direct listening, but **Q4 sounded similar in most segments, with subtle differences audible at the ends of sentences.**

I judged the additional memory gain (about −7 %p going from Q8 to Q4) wasn't enough to justify the perceptual loss, so **I adopted Q8 as the default**.

### 3. Why the Synthesizer Couldn't Be Quantized

The next natural question was, "Then why not quantize the Synthesizer too?" Since that's where all the time goes.

To cut to the chase, **I ended up not applying Synthesizer quantization.** I tried two directions and neither had any meaningful effect.

#### 1. FP16 casting (PyTorch) — Flow layer breaks

When I tried casting the Synthesizer to FP16 in PyTorch, a function called `rational_quadratic_spline` inside the Flow layer broke due to insufficient precision, and with some probability the following assertion would fire:

```
AssertionError: discriminant < 0
```

This function is a **transformation function** that converts inputs to outputs by a fixed rule. During VITS inference, this transformation is called **in reverse (inverse pass)**, and that process uses the **quadratic formula**.

Excerpting from the inverse branch of the original SBV2 [`transforms.py`](https://github.com/litagin02/Style-Bert-VITS2/blob/66de777e06392c0f313600be03c43ef96658b244/style_bert_vits2/models/transforms.py#L167):

```python
# Coefficients of the quadratic equation ax² + bx + c = 0
a = (inputs - input_cumheights) * (
    input_derivatives + input_derivatives_plus_one - 2 * input_delta
) + input_heights * (input_delta - input_derivatives)
b = input_heights * input_derivatives - (inputs - input_cumheights) * (
    input_derivatives + input_derivatives_plus_one - 2 * input_delta
)
c = -input_delta * (inputs - input_cumheights)

discriminant = b.pow(2) - 4 * a * c
assert (discriminant >= 0).all()        # ← breaks here

root = (2 * c) / (-b - torch.sqrt(discriminant))
```

If the discriminant `b² - 4ac` is negative, there are no real roots and the transformation isn't defined, so the code blocks that possibility with `assert`. Mathematically, when inputs and weights are within normal ranges, `≥ 0` is always guaranteed, but **with floating-point, the story changes.** Drop to FP16 and precision is insufficient, causing tiny rounding errors, and as a result `discriminant` ends up negative in some cases, firing the assertion with some probability.

#### 2. INT8 dynamic quantization (ONNX Runtime) — Nothing to quantize

Next I tried ONNX Runtime's dynamic quantization. This approach stores only weights as INT8 and lets activations flow as FP32, so at least the arithmetic inside the spline doesn't break.

But when I tried it, there was another issue. ONNX Runtime's dynamic quantization **only quantizes `MatMul` operations**, and the Synthesizer is mostly `Conv1d`, so there's essentially almost nothing to quantize.

| Model | FP32 | Q8 | Change |
|---|---|---|---|
| BERT (DeBERTa 330M, MatMul-heavy) | 1,159 MB | 544 MB | **−47 %** |
| Synthesizer (Conv1d-heavy) | 239 MB | 239 MB | **0 %** |

I actually quantized the Synthesizer to ONNX Q8 but the model file size stayed the same, and there was barely any change in inference speed.

Additionally, the Synthesizer itself is small at about 63 M — about 1/5 of BERT — so however you quantize it, the memory gain you can get isn't as big as BERT.

So how do I get Synthesizer speed? That question naturally followed, and the answer was ONNX.

### 4. ONNX Runtime Graph Optimization

ONNX Runtime automatically applies **graph-level optimization** when loading a model. Without quantization, it goes through the following transformations to speed up inference.

- **Kernel fusion** — merges multiple consecutive operations into one. For example, when three steps `Conv → BatchNorm → Activation` become one fused kernel, the cost of writing intermediate results to memory and reading them back disappears, saving memory bandwidth.
- **Constant folding** — pre-computes operations that always produce the same value regardless of input at load time. At inference time, the pre-computed values are used as-is.
- **Removing unnecessary nodes** — finds and removes operation nodes that are unused, redundant, or meaningless.

On top of this, ONNX Runtime distributes a single operation across multiple CPU cores via [intra-op parallelism](https://onnxruntime.ai/docs/performance/tune-performance/threading.html). Even with only one concurrent request, you can use the entire CPU, which is advantageous for single-speaker, real-time inference scenarios.

#### Application Results — CPU Speedup

Speedup = audio length / inference time (higher is faster).

| Config | short (1.7s) | medium (7.6s) | long (10.7s) | xlong (38.5s) |
|---|---|---|---|---|
| SBV2 PyTorch FP32 | 1.52× | 2.27× | 2.16× | 1.09× |
| SBV2 ONNX FP32 | 1.76× | 3.09× | 3.26× | 2.75× |
| **HayaKoe (Q8 BERT + FP32 ONNX)** | **2.50×** | **3.35×** | **3.33×** | **3.60×** |

For xlong text (38.5 seconds), the original PyTorch's 1.09× went up to **3.60× in HayaKoe**. From barely keeping up with real-time, it can now process the same input about 3 times faster.

#### Memory — Combined with the BERT Q8 Effect

| Config | RAM (1 speaker) |
|---|---|
| SBV2 PyTorch FP32 | 5,122 MB |
| SBV2 ONNX FP32 | 2,967 MB |
| **HayaKoe (Q8 BERT + FP32 ONNX)** | **2,346 MB** (−54 %) |

ONNX conversion alone reduced RAM by about 42 % (since PyTorch overhead is removed), and BERT Q8 quantization on top of that cut memory further, bringing the final result to −54 %.

### 5. The 89 MB Memory Incident — The Pitfalls of Measurement

All the numbers so far look plausible, but **I went on quite a detour to make those numbers trustworthy.**

When I first wrote the benchmark, it was simple. Within a single Python process, I'd load PyTorch model → ONNX FP32 → ONNX Q8 in order, measuring RAM at each point and comparing.

```python
# Pseudocode
result["pytorch"]      = measure(load_pytorch_model)
result["onnx_fp32"]    = measure(load_onnx_fp32)
result["onnx_all_q8"]  = measure(load_onnx_q8)  # ← 89 MB came out here
```

But while looking at the result JSON, I found a strange value.

```
onnx_all_q8 RAM: 89 MB
```

**89 MB.** No matter how small the Q8 model is, BERT INT8 + Synthesizer FP32 weights alone should add up to nearly 1 GB. When I actually launched the same model as a standalone process, it came out to about 1,757 MB, but in single-process measurement it registered as 89 MB.

#### Tracing the Cause — Python Memory Allocator

It was hard to pinpoint the exact cause, but the hypothesis I came up with based on observed behavior was — **maybe the next model just reused the memory region the previous model had grabbed**.

- Loaded the first model (PyTorch ~2,700 MB) → process RSS climbed to 2.7 GB
- Released the first model with `del` → it disappears from Python's object graph, but from the OS's perspective, the process seems to still hold that region
- Loaded the second model → seems to have reused the previously freed region instead of asking the OS for new memory
- The RSS delta seen by `psutil` was nearly 0 when loading the second model → only the small additional 89 MB was captured

That is, the measurement itself does see "the exact RSS change at that moment," but for the question we wanted to answer — "how much memory does this model use standalone" — it was giving the wrong answer.

I tried forcing release with `gc.collect()` or `del`, but didn't see meaningful differences, which was further circumstantial evidence supporting the hypothesis.

#### Solution — Process Isolation

In the end, **I rewrote the measurement code to run each config as an independent subprocess**. When the PyTorch process ends, the OS reclaims its memory, and the next ONNX process starts from a clean slate.

```python
# Pseudocode
for config in ["pytorch", "onnx_fp32", "onnx_all_q8"]:
    result = subprocess.run(
        ["python", "measure_one.py", "--config", config],
        capture_output=True,
    )
    save_result(config, parse(result.stdout))
```

After this isolation, `onnx_all_q8` RAM measured normally at about 1,757 MB, and that result became the basis for the "2,346 MB / -54 %" number I showed above.

### Wrapping Up Part 2

In Part 2, I summarized how a model that was 5,122 MB · 1.09× was refined down to 2,346 MB · 3.6×.

In summary — **BERT had its memory consumption reduced via Q8 quantization**, **the Synthesizer was converted to ONNX instead of being quantized to get the graph optimization benefit alone**, and finally I spent time **making the measurement environment itself trustworthy**.

This is enough for general usage scenarios, but as you actually synthesize multi-sentence audio, more details start to surface. Things like additional acceleration in GPU environments, multi-sentence BERT batch inference, and the natural pauses that disappear during multi-sentence synthesis.

Continued in [Part 3 — Down to the Last 1%: torch.compile, Batch Inference, Pause Restoration, ARM64](/en/p/9a99dd816456c397/).
