---
title: "hayakoe - Building an Easy-to-Use, Fast TTS Library - Introduction"
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
    - Quantization
    - Python
    - Library
---

_Note: I'm based in Korea, so some context here is Korea-specific._

About two years ago, I came across a Japanese TTS library called Bert-VITS2, and I once wrote a post summarizing how to train it.

After that, I deployed the trained model on my home cluster server and had been using it casually. I forked the original code, then trimmed it down to keep only the path needed for voice synthesis.

But as time went on, small inconveniences started piling up.

- **The `openjtalk` library, which converts Japanese text to pronunciation info, keeps downloading something from external servers, and I don't really know how that works.** I was using it anyway, but I had a nagging feeling that if those download servers ever went down, my server would go down with them.
- **`openjtalk` was blocking Python version upgrades.** Due to dependency compatibility issues, I had been stuck on an old version for a long time.
- **The TTS doesn't support English (it just goes silent).** I worked around it by adding a translation layer in front to convert to Japanese first and then synthesize. It worked, but it was a small but persistent annoyance.
- And the deciding factor: **most of the original libraries had moved on to a new library called fish-speech, and Style-Bert-VITS2 (SBV2), which inherited from it, had stopped being updated 8 months ago.**

> If I'm just going to maintain it anyway, I might as well take this chance to overhaul everything and make it the way I want to use it.

That was the start of this project. And while I was at it, my second goal was to dig deep into the model code itself and study whether there was any room for optimization.

### The Result — `hayakoe`

- **Repository**: [github.com/LemonDouble/hayakoe](https://github.com/LemonDouble/hayakoe)
- **Documentation**: [lemondouble.github.io/hayakoe](https://lemondouble.github.io/hayakoe/)
- **PyPI**: `pip install hayakoe`

It ended up being a library you use roughly like this.

```python
from hayakoe import TTS

tts = TTS().load("jvnv-F1-jp").prepare()
tts.speakers["jvnv-F1-jp"].generate("こんにちは").save("output.wav")
```

In a nutshell, the features:

- **No more `openjtalk` external downloads** — the dictionary is bundled in, so there's no dependency on external servers even on the first run
- **Latest Python support** — install and run directly on the newest Python 3.x environments
- **Direct English synthesis / custom dictionary registration** — built-in 220K-entry foreign-word dictionary auto-converts English to katakana, and you can register pronunciations directly for unknown proper nouns
- **CPU real-time inference** — 1.5x to 3.6x faster than PyTorch, with about half the RAM (5,122 MB → 2,346 MB, -54%)
- **No torch needed** — runs without PyTorch for CPU inference, keeping dependencies light
- **Concise API** — start with one line: `TTS().load(...).prepare()` / HuggingFace transformers style
- **Pluggable sources** — mix `hf://`, `s3://`, `file://` in a single instance
- **FastAPI server integration** — supports both sync and async handlers
- **ARM64 support** — runs on Raspberry Pi 4B too, though it's slow

### "So what did you actually do to make it faster?"

I started off thinking something like "maybe I'll just try quantization."

Looking at weight file sizes, BERT took up the largest share of the model. So I thought "I just need to quantize this, right?" — but when I actually measured it, that wasn't the case.

- The longer the text, **the more the Synthesizer dominates CPU time at 80–91%**, and BERT quantization only reduced less than 5% of total time.
- BERT INT8 quantization had its real value not in speed but in **memory savings (1,698 MB → 368 MB, -78%)**.
- "Then I should just quantize the Synthesizer" — but the Synthesizer itself can't be quantized: the `rational_quadratic_spline` inside the Flow layer breaks due to numerical instability starting from FP16.

In other words, I once again learned the lesson that if you don't measure precisely where the actual bottleneck is, you end up pouring time into the wrong places.

So in the end, the work flowed roughly like this.

1. **Re-picking the model** — two years had passed, and since I was rebuilding it anyway, I figured I'd look around at other libraries. I evaluated GPT-SoVITS, which is said to have good Korean support, and autoregressive ones (Qwen-3-TTS, etc.). Since latency was a fairly important factor for real-time alarms/briefings, I ended up settling on Style-Bert-VITS2 (JP-Extra v2.7.0), which had the best balance of quality and speed.
   - **GPT-SoVITS** — real-time speed was satisfying, but the perceived quality of inferred voice was lower than SBV2, so dropped.
   - **Qwen-TTS** — too computationally heavy. My existing GPU was an RTX 2070 Super, and since flash-attn is only supported on RTX 3000-series and above, I thought "is this just because flash-attn isn't installed?" and even bought a new RTX 3080. But generation time was still too long, so I concluded real-time use was impractical.
2. **Different prescriptions per component** — based on measurements, I applied different optimizations to BERT and the Synthesizer.
   - **BERT** — used INT8 dynamic quantization to reduce memory only. Doesn't impact speed much, but memory matters a lot when running multiple programs on one server.
   - **Synthesizer** — converted to ONNX and applied graph-level optimizations. ONNX Runtime automatically applies optimizations like kernel fusion (combining multiple operations into one) and constant folding (precomputing constant operations at load time).

3. **Polishing the small details** — beyond the main thread, I cleaned up the smaller items too.
   - **`torch.compile`** — additional speedup via PyTorch JIT compilation on GPU inference.
   - **BERT batched inference** — bundling BERT calls into one when synthesizing multiple sentences, reducing overhead.
   - **Restoring natural pauses** — original SBV2 has natural pauses after punctuation, but they disappear when you split-synthesize multiple sentences. I separately exported just the Duration Predictor to ONNX and used it to predict pause lengths at punctuation positions.
   - **ARM64 build** — pre-built so it runs on ARM devices like Raspberry Pi 4B too.

4. **Designing it as a library** — I cared about not just making it work but making it easy for others to pick up and use.
   - **Per-speaker memory separation** — shared resources like BERT and WavLM are loaded once, and each speaker only holds their own weights. Memory doesn't explode as speakers grow.
   - **multi-source support** — mix `hf://`, `s3://`, `file://` in one instance. Use HuggingFace for official models, S3 if you're already on S3, local files for in-development models, etc.
   - **Thread-safe singleton serving** — safe to share one TTS instance across multiple handlers in a FastAPI server.
   - **Docker build pattern** — bake the model into the image at build time, and load instantly from cache at runtime.

### Posts to Come

I'm planning to split this series into 4 parts.

1. **(Part 1) Coming Back to TTS Models After 2 Years and Settling on VITS Again**
   Why I tried GPT-SoVITS v4 and switched away, how I gathered the training dataset, and how I picked the optimal step via blind listening tests of 10 checkpoints.
2. **(Part 2) Half the Memory, 1.5x the Speed**
   Bottleneck measurement methodology, quantization experiments (why the Synthesizer can't be quantized), ONNX conversion, and the time memory was incorrectly measured at 89MB.
3. **(Part 3) The Last 1% — torch.compile, Batched Inference, Pause Restoration, ARM64**
   GPU inference polish and the story of separately exporting just the Duration Predictor to ONNX.
4. **(Part 4) Things I Thought About While Packaging It as a Library**
   API design philosophy, multi-source (`hf://` / `s3://` / `file://`), thread-safe singleton serving, FastAPI / Docker deployment.

### Wrapping Up the Introduction

When I wrote my first training post, I was thinking "I'll just train this once and be done." But once I actually had the model in hand, **how to make it easy for others to pick up and use** turned out to be a much bigger task than the training itself.

Along the way, I ended up stepping into areas I don't usually deal with — quantization, ONNX graphs, ML serving — and since I'd done the work, I thought writing it up as a series might help others walking a similar path.
