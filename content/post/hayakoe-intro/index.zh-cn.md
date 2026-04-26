---
title: "hayakoe - 打造易用且快速的TTS库 - 序论"
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
    - 库
---

_注：作者居住在韩国，部分内容包含韩国特有的背景。_

大约两年前，我接触到了一个叫做Bert-VITS2的日语TTS库，并曾经写过一篇整理学习方法的文章。

之后，我把训练好的模型部署到家用集群服务器上日常使用。我对原始代码进行了Fork，然后只把语音合成所需的path抽取出来整理后使用。

但随着时间推移，一些小麻烦渐渐积累起来。

- **将日语文本转换为发音信息的`openjtalk`库，会持续从外部下载一些东西，但其实我也搞不太清楚它具体是怎么运作的。** 虽然一直在用，但总有种"万一那个下载服务器挂了，我的服务器岂不是也跟着挂"的不安感。
- **由于`openjtalk`，Python版本升级被卡住了。** 因为依赖兼容性问题，长时间没能升级，一直放置着。
- **TTS不支持英语（变成无声）。** 我用了在前面加一层翻译层先转成日语再合成的方式绕过去。虽然能跑，但小小的别扭一直存在。
- 而决定性的一点是，**大部分原有库都已经迁移到了一个叫fish-speech的新库，而继承自它的Style-Bert-VITS2 (SBV2) 也在8个月前停止了更新。**

> 反正都要维护，不如借这个机会全部推倒重来，做成自己用着舒服的样子。

这就是这个项目的开始。而且既然要动手，那就顺便深入研究模型代码，看看有没有优化的余地——这是第二个目的。

### 成果 — `hayakoe`

- **Repository**: [github.com/LemonDouble/hayakoe](https://github.com/LemonDouble/hayakoe)
- **Documentation**: [lemondouble.github.io/hayakoe](https://lemondouble.github.io/hayakoe/)
- **PyPI**: `pip install hayakoe`

最终大致变成了这样使用的库。

```python
from hayakoe import TTS

tts = TTS().load("jvnv-F1-jp").prepare()
tts.speakers["jvnv-F1-jp"].generate("こんにちは").save("output.wav")
```

特性总结一下：

- **去除`openjtalk`外部下载** — 词典一并打包，首次运行也无需依赖外部服务器
- **支持最新Python** — 在最新Python 3.x环境直接安装/运行
- **直接合成英文输入 / 自定义词典注册** — 内置22万条目外来语词典自动转换英文→片假名，未知专有名词可直接注册发音
- **CPU实时推理** — 比PyTorch快1.5×～3.6×，RAM减少约一半（5,122 MB → 2,346 MB，-54%）
- **无需torch** — CPU推理无需PyTorch即可运行，依赖更轻量
- **简洁API** — `TTS().load(...).prepare()`一行即可启动 / HuggingFace transformers风格
- **可插拔的源** — 可在一个实例中混用`hf://`, `s3://`, `file://`
- **FastAPI服务器集成支持** — 同时支持同步/异步处理器
- **ARM64支持** — 可在Raspberry Pi 4B上运行，但速度较慢

### "那你到底做了什么让它变快的？"

最初只是抱着"试试量化吧"这种轻松的想法开始的。

光看weight文件大小，BERT在模型中占的比重最大。所以我以为"只要量化这个就行了吧？"，但实际测量后发现并非如此。

- 文本越长，**Synthesizer占用CPU时间的80～91%**，BERT量化能减少的时间还不到整体的5%。
- BERT INT8量化的实际价值不在速度，而在于**节省内存（1,698 MB → 368 MB，-78%）**。
- "那量化Synthesizer不就行了"——但Synthesizer本身就无法量化，因为Flow层中的`rational_quadratic_spline`从FP16开始就会因数值不稳定而崩坏。

也就是说，如果不精确测量真正的瓶颈在哪里，就会把时间花在错误的地方——这又是一次重新学到的教训。

所以最终工作大致按这个流程进行。

1. **从模型开始重新选择** — 既然过了两年，反正都要重做，不如顺便看看其他库。我把据说韩语支持很好的GPT-SoVITS、自回归类（Qwen-3-TTS等）作为候选进行了评估。由于是实时闹钟/简报用途，延迟时间相当重要，最终落定在音质和速度平衡最好的Style-Bert-VITS2 (JP-Extra v2.7.0)。
   - **GPT-SoVITS** — 实时速度令人满意，但推理语音的体感品质不如SBV2，淘汰。
   - **Qwen-TTS** — 计算量太大被淘汰。我原本的GPU是RTX 2070 Super，得知flash-attn只支持RTX 3000系以上的显卡后，就想"是不是没装flash-attn导致的？"，甚至专门买了RTX 3080，但生成时间还是太长，最终得出实时使用不现实的结论。
2. **针对各组件应用不同的处方** — 基于测量结果，对BERT和Synthesizer分别应用了不同的优化。
   - **BERT** — 用INT8 dynamic quantization只减少内存。对速度影响不大，但要在一台服务器上跑多个程序时，内存挺重要的。
   - **Synthesizer** — 转换为ONNX并应用图级优化。ONNX Runtime会自动应用kernel fusion（将多个运算合并为一个）、constant folding（在加载时预先计算常量运算）这类优化。

3. **完成琐碎的细节** — 主线之外的细节也一并整理。
   - **`torch.compile`** — 在GPU推理上通过PyTorch JIT编译进一步提速。
   - **BERT批推理** — 多句合成时把BERT调用打包成一次，减少开销。
   - **恢复自然停顿** — 原版SBV2在标点后会有自然的停顿，但分割合成多句时这些停顿就消失了。我把Duration Predictor单独export成ONNX，让它预测标点位置的停顿长度。
   - **ARM64构建** — 预先构建以便在Raspberry Pi 4B这类ARM设备上也能运行。

4. **作为库的设计** — 不只是让它能动，而是要让别人也能轻松拿来用，这部分我下了不少功夫。
   - **按说话人分离内存** — BERT、WavLM这类共享资源只加载一次，每个说话人只持有自己的权重。说话人增加内存也不会爆炸。
   - **multi-source支持** — 一个实例可以混用`hf://`, `s3://`, `file://`。官方模型用HuggingFace，已经在用S3就用S3，开发中的模型用本地，等等。
   - **Thread-safe单例服务** — 在FastAPI服务器中，多个处理器共享一个TTS实例也是安全的。
   - **Docker构建模式** — 构建时把模型打入镜像，运行时从缓存即时加载。

### 接下来要写的文章

这个系列计划分为4部分。

1. **(Part 1) 时隔2年再次尝试各种TTS模型，又一次落定到VITS的故事**
   尝试GPT-SoVITS v4后为什么换掉了，学习数据集是怎么收集的，10个checkpoint的听感评估如何确定最佳step。
2. **(Part 2) 内存减半、速度提升1.5倍的故事**
   瓶颈测量方法论、量化实验（为什么Synthesizer无法量化）、ONNX转换，以及内存被错误测量为89MB的那次事件。
3. **(Part 3) 直到最后1% — torch.compile、批推理、Pause恢复、ARM64**
   GPU推理的精修，以及单独把Duration Predictor抽出来export成ONNX的故事。
4. **(Part 4) 打包成库时考虑的事情**
   API设计哲学、multi-source（`hf://` / `s3://` / `file://`）、thread-safe单例服务、FastAPI / Docker部署。

### 序论结语

最初写学习文章的时候，我还想着"训练这一次就完事了"，但等模型真的拿到手，**怎么让别人也能轻松拿去用**才是比训练本身大得多的工程。

在这个过程中我也踏入了平时不太接触的量化、ONNX图、ML服务领域，既然都做了，整理下来或许对走类似道路的人也有帮助，所以打算以系列形式留下记录。
