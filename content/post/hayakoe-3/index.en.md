---
title: "Squeezing Out the Last 1% — torch.compile, Batch Inference, Pause Restoration, ARM64 - hayakoe Part 3"
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
    - Optimization
    - PyTorch
    - Raspberry Pi
    - Python
---

_Note: I'm based in Korea, so some context here is Korea-specific._

This is the third main installment of the hayakoe series. You can find the previous post at [Halving Memory and Boosting Speed by 1.5× - hayakoe Part 2](/en/p/137243ede98030f1/).

In [Part 2](/en/p/137243ede98030f1/), I made significant improvements to memory and speed via the main pillars — BERT Q8 quantization + Synthesizer ONNX conversion. But once I started actually running this model in production, I noticed a few things that felt unsatisfying, separate from the main pillars. They were the kind of work where I thought it would be nice to polish things into a more usable form.

In Part 3, I cover the four areas I touched up:

1. **`torch.compile`** — GPU inference acceleration
2. **BERT GPU retention + batch inference** — meaningful difference in multi-sentence synthesis
3. **Natural pause restoration** — bringing back the post-punctuation pauses that disappear when synthesizing multi-sentence text in chunks
4. **ARM64 build** — making it run on Raspberry Pi 4B

### 1. `torch.compile` — GPU inference acceleration

`torch.compile`, introduced in PyTorch 2.0, JIT-compiles the model graph (compiling dynamically at runtime) to gain extra speed. It uses CUDA Graphs (a feature that bundles repeated GPU operation sequences into a single replayable unit) to reduce GPU call overhead, and where possible, it fuses multiple operations into a single GPU kernel (fused kernels).

In hayakoe, when `prepare()` is called and the device is CUDA, `torch.compile` is automatically applied — from the user's perspective, you just turn on GPU mode and it works without any extra configuration.

| Backend | Short sentence | Medium sentence | Long sentence |
|---|---|---|---|
| PyTorch (CUDA) | 7.3× | 16.3× | 13.6× |
| `torch.compile` | 7.4× | 17.2× | **15.4×** |
| Improvement | +1 % | +6 % | **+13 %** |

The reason long sentences see a bigger improvement is that the longer the text, the more Conv kernels the Synthesizer calls, and the launch overhead accumulates accordingly. CUDA Graphs absorbs all that overhead in one go.

However, getting this benefit requires **warmup**. CUDA Graphs takes time to capture and compile the graph, so the first few calls are actually slower. hayakoe runs about 8 dummy inferences via the `prepare(warmup=True)` option, so users don't see compilation cost on their first request.

### 2. BERT GPU retention + batch inference

I also addressed two things that were eating away at GPU path efficiency — **unnecessary GPU↔CPU round trips** and **per-sentence individual BERT calls**.

#### Removing `.cpu()` — keeping tensors on GPU

The original SBV2's [BERT feature extraction code](https://github.com/litagin02/Style-Bert-VITS2/blob/66de777e06392c0f313600be03c43ef96658b244/style_bert_vits2/nlp/japanese/bert_feature.py#L61) had this section:

```python
# Original SBV2 (style_bert_vits2/nlp/japanese/bert_feature.py)
res = torch.cat(res["hidden_states"][-3:-2], -1)[0].cpu()
```

After running BERT forward on GPU, it was calling `.cpu()` on the output to bring it down to CPU every time. But this output is then immediately handed to the Synthesizer (also on GPU), so it has to be moved back to GPU. The result is a **GPU → CPU → GPU round trip** for every sentence, and that round trip itself becomes a small bottleneck.

I changed the original code as follows so the BERT output stays as a GPU tensor:

```python
# hayakoe
res = torch.cat(res["hidden_states"][-3:-2], -1)[0].float()  # keep on GPU
```

The reason I call `.float()` instead of `.cpu()` is dtype unification (casting between FP16 BERT and FP32 Synthesizer); the details are covered in [the BERT quantization / FP16 casting section of Part 2](/en/p/137243ede98030f1/).

Additionally, I manage the BERT model itself as a **global singleton**, so even when there are multiple speakers, BERT is loaded onto the GPU only once and all speakers share that instance.

#### Batching BERT for multi-sentence inputs

For prosody stability, hayakoe splits input text by punctuation and synthesizes sentence by sentence. As a natural consequence, BERT ends up being called as many times as there are sentences.

On GPU, each operation call carries a fixed cost (kernel launch overhead), so when sentences are short, this call cost can exceed the actual computation time, and the inefficiency accumulates. Fortunately, BERT (DeBERTa) is a HuggingFace Transformer that natively supports batched input, so I **bundled all sentences into a single batch and call BERT forward only once**.

| Sentences | Sequential | Batched | Speedup |
|---|---|---|---|
| 2 | 0.447 s | 0.364 s | **1.23×** |
| 4 | 0.812 s | 0.566 s | **1.43×** |
| 8 | 1.598 s | 1.121 s | **1.43×** |
| 16 | 2.972 s | 2.264 s | **1.31×** |

As the number of sentences grows, a stable +23% to +43% speedup shows up. The memory difference was within 1.3 MB, so practically identical.

Interestingly, repeating the same experiment on CPU (ONNX) shows almost no effect. Measurements show only noise-level differences between +1% and −10%.

Since there's no big effect on CPU but also no real loss, I kept batching on so that GPU and CPU run through the same code path.

### 3. Natural pause restoration

This is the most detailed part of this installment.

#### Side effect of split synthesis

As mentioned above, hayakoe splits multi-sentence input by punctuation and synthesizes each sentence separately, for prosody stability. When you try to synthesize long text all at once, intonation tends to get muddled or unstable, so I introduced this structure for stability and naturalness.

But this split has one side effect: **the natural pauses between sentences disappear**.

The original SBV2 produces natural pauses after punctuation marks like `.`, `!`, `?` in whole-text synthesis. But when split sentence by sentence, each sentence ends at the punctuation and the next sentence starts from scratch, so the post-punctuation pauses also disappear. In my initial implementation, I tried inserting a fixed 80 ms silence between sentences, but real natural pauses are around 0.3 to 0.6 seconds, so 80 ms was way too short, resulting in unnaturally rushed speech that felt "out of breath."

#### How did the original SBV2 produce pauses?

I traced how the original SBV2 produces natural pauses in whole-text synthesis. The conclusion was surprisingly simple — it was a **side effect of the Duration Predictor predicting frame counts for punctuation phonemes**.

The Duration Predictor is originally a module that predicts "how many frames each phoneme should be pronounced for." Like 5 frames for "an" and 4 frames for "nyung." But punctuation marks like `.`, `!`, `?` are also included in the phoneme sequence, and the Duration Predictor predicts frame counts for these punctuation phonemes too. The predicted frame count becomes the pause length at that position.

In split synthesis, this information was being discarded because synthesis was cut off at the punctuation positions.

#### Solution — running just the Duration Predictor separately

With the problem and cause both clear, the solution followed naturally.

The core idea is simple. **Pass the original pre-split text only through TextEncoder + Duration Predictor to get the frame counts at punctuation positions in advance**. Skip Flow and Decoder (the parts that actually generate audio).

```
Full text (pre-split original)
  │
  ├─ TextEncoder (G2P → phoneme sequence → embedding)
  │
  ├─ Duration Predictor (predicts frame counts per phoneme)
  │     └─ Extract only frame counts at punctuation positions
  │
  └─ Compute pause time
        frames × hop_length / sample_rate = seconds
```

Most of the cost of a full Synthesizer pass is in Flow + Decoder (see [Part 2's bottleneck measurements](/en/p/137243ede98030f1/)), so the cost of running just up to the Duration Predictor is very low compared to full synthesis.

With hayakoe's default settings of `hop_length = 512` and `sample_rate = 44100`, 1 frame corresponds to about 11.6 ms, so if the combined frame count of the punctuation + adjacent blank token is 35:

```
35 × 512 / 44100 ≈ 0.41 seconds
```

This is how I get a natural pause time at each sentence boundary.

#### ONNX support — exporting just the Duration Predictor separately

In the PyTorch path, I can call individual modules of the model directly, so I just pick out and run only the Duration Predictor. But `synthesizer.onnx` exports the whole Synthesizer as a single end-to-end graph, so it's impossible to extract just the Duration Predictor output mid-graph.

To solve this, I additionally exported a **separate ONNX model containing only TextEncoder + Duration Predictor**.

- Artifact: `duration_predictor.onnx` (~30 MB, FP32)
- Runs on ONNX Runtime
- If this file is missing in existing deployment models, it silently falls back to 80 ms (backward compatible)

#### Results

For the same text, the auto-predicted sentence boundary pauses:

| Backend | Pause range |
|---|---|
| GPU (PyTorch) | 0.41 s ~ 0.55 s |
| CPU (ONNX) | 0.38 s ~ 0.57 s |

The difference between the two backends falls within the natural variation that arises from the SDP (Stochastic Duration Predictor)'s probabilistic sampling. In other words, there's effectively no quality loss from ONNX conversion.

Listening to multi-sentence samples directly, the difference is quite clear. The "Before" with 80 ms fixed silence sounds like the sentences are running into each other, while the "After" with Duration Predictor-predicted pauses sounds closer to the breathing flow of human speech.

### 4. ARM64 build — Raspberry Pi 4B

Finally, I made hayakoe **work on aarch64 (ARM64) Linux with the same code**, not just x86_64.

This was possible because of two conditions:

- **ONNX Runtime** officially provides aarch64 builds
- **`pyopenjtalk`**'s own fork ([`lemon-pyopenjtalk-prebuilt`](https://github.com/LemonDoubleHQ/lemon-pyopenjtalk-prebuilt), [related post](https://blog.lemondouble.com/p/pyopenjtalk-prebuilt/)) builds aarch64 wheels and bundles the dictionary into the package

The second condition is also a natural byproduct of solving the "external `openjtalk` download dependency" problem mentioned in the [introduction](/en/p/ea78fbb9b78c38ac/).

#### Raspberry Pi 4B real measurements

Measurements on a Raspberry Pi 4B (Linux 6.8, aarch64, ONNX Runtime 1.23.2):

| Text | Inference time | Speed |
|---|---|---|
| Short | 3.169 s | 0.3× |
| Medium | 13.042 s | 0.3× |
| Long | 35.119 s | 0.3× |

At about 1/3 of real time, it's not enough for conversational use. But I think there's value in just being able to run it on an ARM board — it's perfectly usable for offline batch synthesis or asynchronous tasks running on ARM nodes in a cluster.

> I expect it would also work on Apple Silicon (macOS), but I haven't been able to verify since I don't have the test hardware.

### Wrapping up Part 3

In Part 3, I covered **four lawn-trimming-level details** beyond the major optimization pillars.

I picked up another +13% on long sentences with `torch.compile`, fixed inefficiencies in the GPU path with BERT GPU retention and batch inference, restored the naturalness of multi-sentence split synthesis by separating out the Duration Predictor, and finally extended the operating range to Raspberry Pi 4B.

With this, optimization on the model and inference side is essentially complete. In the final Part, I'll cover how I packaged this into a library that others can easily pick up and use — API design, multi-source, thread-safe singleton serving, FastAPI / Docker patterns.

Continued in [Part 4 — Things I considered while packaging into a library](/en/p/b7791b4f6c0818da/).
