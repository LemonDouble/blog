---
title: "hayakoe — Two Years Later, Surveyed TTS Models Again and Still Landed on VITS — Part 1"
slug: 4c2a6225a737f531
date: 2026-04-26T16:33:46+09:00
image: "cover.png"
categories:
    - AI
    - TTS
tags:
    - AI
    - TTS
    - Style-Bert-VITS2
    - GPT-SoVITS
    - Voice Synthesis
    - Python
---

_Note: I'm based in Korea, so some context here is Korea-specific._

This is the first proper entry in the hayakoe series. The intro post is over at [hayakoe — Building a TTS Library That's Easy and Fast to Use — Intro](/en/p/ea78fbb9b78c38ac/).

One topic I touched on briefly in the intro was "I picked the model from scratch again." In Part 1, I want to walk through that process in more detail.

Roughly speaking, it breaks down into three steps:

1. Surveying other library candidates and circling back to SBV2
2. Building the dataset (the tooling has come a long way in two years)
3. Training setup and checkpoint evaluation

### 1. Looking Around at Other Libraries

Two years ago, when I wrote [Building My Own TTS with Bert-VITS2](/en/p/9c06f7dc324b5986/), Bert-VITS2 was pretty much the only realistic option for a Japanese TTS library. Two years on, the voice synthesis space has moved quite a bit.

If I was going to spend time on this anyway, it made sense to look around at other options. So I evaluated two candidates that supposedly had good Korean support.

#### GPT-SoVITS

The first one I looked at was GPT-SoVITS. It supports Korean / English / Japanese, and the appeal was zero-shot inference — feed a pretrained model a short sample (a few seconds) and it produces speech in that voice.

I tried it zero-shot first, and the result was unsatisfying on two fronts:

- **Speech naturalness wasn't there.** The synthesized speech sounded flat, with awkward transitions in emotion and intonation.
- **Korean output was off.** When I fed it a Japanese speaker's sample and made it speak Korean, the result was — "technically Korean," but not natural. It felt like a Japanese speaker forcing themselves through Korean syllables..?

If I already wasn't satisfied with the zero-shot quality, training one more time wasn't going to magically fix it. And since what I actually wanted was Japanese output anyway, I figured it made more sense to look at more candidates instead of running training. So I moved on without training.

#### Qwen-TTS / Autoregressive Family

As I briefly mentioned in the intro, the autoregressive Qwen-TTS was also a candidate. The first deal-breaker here was that the compute was too heavy for real-time use, before I even got to audio quality (see the [intro post](/en/p/ea78fbb9b78c38ac/) for details).

The audio quality wasn't great either. Maybe because it was zero-shot, the level of detail I wanted just wasn't there. ~~If you're enough of an otaku, you know what I mean.~~

#### Back to Style-Bert-VITS2

After looking at the candidates, I came back to Style-Bert-VITS2 (JP-Extra v2.7.0) — it had the best balance of audio quality and speed. SBV2 is the successor to Bert-VITS2, and the fact that I'd already gone through training/usage with it two years ago helped a lot.

### 2. Building the Dataset — Things Got a Lot Better in Two Years

For the dataset, I had personal voice data I'd already collected, and I just used that. I'd done similar work two years ago, so the overall flow was familiar. After cleanup and quality validation, I ended up with around 1,500 samples.

What stood out was how much the voice processing tooling has evolved in two years. The most striking thing was that vocal separation tools (UVR — Ultimate Vocal Remover) no longer require installing a separate GUI application.

Two years ago, this was a bit of a hassle. UVR was a Windows app, so you had to install it directly, then click "Download Model" inside the app to get the model files separately, and only then could you run folder-level batch processing. Now, with libraries like `audio-separator`, you can pull the model from HuggingFace and run it with one line of Python.

```python
from audio_separator.separator import Separator

separator = Separator()
separator.load_model("model_bs_roformer_ep_317_sdr_12.9755.ckpt")
separator.separate("input.wav")
```

This might sound like a minor difference, but when you're processing 1,500 samples, cutting out the GUI clicks / folder management / progress checks adds up. More importantly, you can fold this directly into the training pipeline, so when you add new data, it's just running the same code again.

### 3. Training Style-Bert-VITS2

For training itself, I just used the webui that ships with SBV2.

- **Sampling rate**: 44.1 kHz
- **batch_size**: 2 (RTX 3080 / 10GB VRAM)
- **learning rate**: 1e-4
- **eval_interval**: 5,000 steps
- **Target epochs**: 500 (actually stopped around epoch 88)
- **Training data**: ~1,500 samples

There was no real need to run all 500 epochs. After a certain step count, audio quality stopped improving — or started showing signs of overfitting — so I stopped at a reasonable point and moved to evaluation.

### 4. Checkpoint Evaluation — I Just Listened to All of Them

The most accurate way to know how audio quality changes across checkpoints is, well, to listen to them. SBV2 does log various losses to tensorboard during training — mel reconstruction loss, KL divergence, duration loss, generator / discriminator loss, WavLM adversarial loss, and so on. But lower numbers don't necessarily translate into more natural speech. In the end, what mattered was whether it sounded right to my ears.

Since I was the only one going to use the model, I kept evaluation simple.

- Pulled **10 checkpoints** from 5,000 to 50,000 steps in 5,000-step increments
- Picked **3 texts** — short, medium, and long — and synthesized each with every checkpoint
- Laid out the 30 resulting samples in an HTML matrix to compare on a single page

```
              short    medium    long
ckpt 5k       ▶ wav    ▶ wav     ▶ wav
ckpt 10k      ▶ wav    ▶ wav     ▶ wav
ckpt 15k      ▶ wav    ▶ wav     ▶ wav
ckpt 20k      ▶ wav    ▶ wav     ▶ wav
...
ckpt 50k      ▶ wav    ▶ wav     ▶ wav
```

Each cell holds the synthesized wav, so you can compare the same text across columns or different steps within a row. The synthesis itself took some time, but the evaluation was just listening through with headphones in order — a mechanical task.

After listening through everything, **15,000 steps (~epoch 34)** sounded the most natural. Beyond that point, the timbre started shifting subtly or the speech tone got monotone — that step was the sweet spot.

> This is purely my ear. Someone else might prefer a different step, and the result might shift slightly with different texts. But since this is a model only I'll be using, this was good enough.

### Wrapping Up Part 1

In Part 1, I covered the reasoning for coming back to Style-Bert-VITS2 (JP-Extra v2.7.0) after looking at other candidates, and then how I worked through dataset / training / checkpoint evaluation on top of that.

To summarize — the other candidates (GPT-SoVITS, Qwen-TTS, etc.) either didn't deliver the detail I wanted at zero-shot quality, or were too compute-heavy for real-time. SBV2 had the best balance of audio quality, speed, and familiarity, and that's where I landed.

That said, the trained SBV2 model still has limits like 5 GB RAM and 1.09× CPU inference. The next post is about how I cut that down.

Continued in [Part 2 — Halving Memory and Hitting 1.5× Faster Inference](/en/p/137243ede98030f1/).
