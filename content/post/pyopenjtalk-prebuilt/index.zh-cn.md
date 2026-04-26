---
title: "想升级个人服务器的 Python 版本，但上游停止更新，于是 fork 了 prebuilt wheel 包让它支持新 Python 版本的故事 (Pyopenjtalk)"
slug: pyopenjtalk-prebuilt
date: 2026-04-05T21:22:00+09:00
image: cover.png
categories:
    - Python
    - CI/CD
tags:
    - Python
    - PyPI
    - wheel
    - cibuildwheel
    - GitHub Actions
    - pyopenjtalk
    - TTS
---

_注：作者居住在韩国，部分内容包含韩国特有的背景。_

之前我写过一篇 [用 Bert-VITS2 做 TTS 的文章](../make-vits2-tts/)，那时候做的 TTS 一直用得挺好。但是…Bert-VITS2 的仓库已经停止开发了。

仓库停止维护会有什么问题呢？升级 Python 版本变得很麻烦，依赖管理也很麻烦。

我的情况是，在家用服务器上有个叫 ML Server 的地方，把所有 ML 相关的功能全塞在那里。所以装了一堆乱七八糟的东西…如果有那种不更新的仓库存在，每次装东西都会冲突。比如 numpy 2.0 之类的…

所以哪怕只是想加一个东西…每次装东西都冲突，搞得很不容易。我就开始追查"到底是谁老在搞坏依赖？"，结果发现就是那个包。不让我升级到 Python 3.10 以上的家伙…我也想尝尝 3.12 的味道啊。

于是忍了又忍…我决定自己干。看到不让我升级到 Python 3.10 以上的原因是 **pyopenjtalk** 之后，我心想：啊，只要把那个搞定就行了！

不过这家伙是把 C 模块编译成 wheel 的，要动它有点麻烦…但是？反正？我搞定了。


## pyopenjtalk?

[pyopenjtalk](https://github.com/r9y9/pyopenjtalk) 是日语 TTS 引擎 [OpenJTalk](https://open-jtalk.sourceforge.net/) 的 Python 封装。它能把日语文本转成语音，或者转成发音符号。

```python
import pyopenjtalk

# TTS: 文本 → 语音波形
x, sr = pyopenjtalk.tts("おめでとうございます")

# G2P: 文本 → 发音
pyopenjtalk.g2p("こんにちは")           # → 'k o N n i ch i w a'
pyopenjtalk.g2p("こんにちは", kana=True) # → 'コンニチワ'
```

内部用 Cython 绑定了 C/C++ 写成的 OpenJTalk，并用 CMake 构建原生代码。

## 问题：安装太难了

pyopenjtalk 在 PyPI 上 **只发布了源码分发版（sdist）**。运行 `pip install pyopenjtalk` 时，需要在用户环境直接编译 C++ 代码。所以安装前必须准备好这些：

- CMake
- C++ 编译器（gcc、MSVC 等）
- Cython

本地开发环境还能凑合，但…在 Docker 镜像、CI 环境或没有构建工具的服务器上，安装就不容易了。

为了解决这个问题，曾经有个叫 [pyopenjtalk-prebuilt](https://pypi.org/project/pyopenjtalk-prebuilt/) 的包，Bert-VITS2 就是用的它。我记得 GPT-SOVITS 也是…在日语 TTS 相关项目中时不时能看到。
它提供预先构建好的 wheel，可以不用构建工具直接安装，但是…**在 Python 3.11 停止更新了。** Python 3.12 以上？用不了。

pyopenjtalk 原版超过一年没更新，prebuilt 也不发新的…

反正代码已经冻结了，那直接 fork 来用不就行了？<<

于是最后我直接 fork 自己做了一个。


## 做了什么

我 fork 了原版 pyopenjtalk 仓库，从零搭建了构建和发布 prebuilt wheel 的流水线。包名叫 `lemon-pyopenjtalk-prebuilt`。

工作大致分四块：

1. 重新整理包配置
2. 搭建 CI/CD 流水线
3. 稳定发布流水线（踩坑）
4. 添加自动检测 Python 新版本的工作流

### 1. 重新整理包配置 (`pyproject.toml`)

我把原版 pyopenjtalk 的 `pyproject.toml` 整理成符合新包的样子。

- **包名**：`pyopenjtalk` → `lemon-pyopenjtalk-prebuilt`
- **Python 最低版本**：3.8 → 3.9
- **numpy**：构建用 `numpy>=2.0`，但实际使用时 `numpy>=1.25.0` 就行（向后兼容）
- **构建目标 Python**：3.8 ~ 3.10 → 3.9 ~ 3.13
- **支持的 OS**：Linux、macOS、Windows → Linux (x86_64)、Windows (AMD64)

然后把不用的文件全删了…

- `docs`、`dev` optional-dependencies — Sphinx（生成文档）、pysen（linter）、mypy（类型检查）等开发工具。我们只需要发布构建好的 wheel，不需要这些
- 整个 pysen linter 配置 — 原仓库的代码风格检查配置。fork 没必要管这个。都停更一年多了…应该没事吧
- Python 3.8 相关的条件依赖 — `importlib_resources`（3.8 用来访问资源）、`oldest-supported-numpy`（3.8 用的 numpy）等。最低版本提到 3.9 后就用不上了

然后核心的 **[cibuildwheel](https://cibuildwheel.pypa.io/)** 配置加上了。这是 PyPA 维护的工具，在 `pyproject.toml` 里声明类似 "CPython 3.9~3.13 × Linux/Windows" 这样的组合，GitHub Actions 就会为每个组合自动构建 wheel。不用自己在每个 OS 上装 Python 跑构建脚本。

这样一次工作流运行就能产出 **10 个 wheel + 1 个 sdist**（Linux 5 个 + Windows 5 个）。

版本管理用 [setuptools_scm](https://github.com/pypa/setuptools-scm)，配置成自动从 git tag 取版本号。

### 2. 搭建 CI/CD 流水线 (`build_and_release.yml`)

原仓库的 CI 是源码构建 + 测试 + sdist 发布的结构。把这个删掉，重新写了基于 cibuildwheel 的 wheel 构建 + 发布工作流。

```
触发：版本 tag (v*) push 或手动运行

┌─────────────────┐   ┌──────────────┐
│ build_wheels     │   │ build_sdist  │
│ (ubuntu, windows)│   │ (ubuntu)     │
│ cibuildwheel     │   │ python -m    │
│ → .whl 工件      │   │   build      │
└────────┬────────┘   │ → .tar.gz    │
         │            └──────┬───────┘
         └───────┬───────────┘
                 ▼
         ┌──────────────┐
         │ publish       │
         │ twine upload  │
         │ → PyPI        │
         └──────────────┘
```

[twine](https://twine.readthedocs.io/) 是把构建好的 wheel 或 sdist 上传到 PyPI 的 CLI 工具。运行 `twine upload dist/*` 就能把构建产物上传到 PyPI。

GitHub Actions 在 checkout 时，有两个选项很关键：

- **`submodules: recursive`** — pyopenjtalk 内部把 OpenJTalk C++ 源码作为 git submodule。没有这个选项的话子模块文件夹会被空着 checkout，没有 C++ 源码可以构建，理所当然会失败。
- **`fetch-depth: 0`** — 默认情况下 GitHub Actions 只拉最新一个提交（shallow clone），但 setuptools_scm 是看 git tag 历史来决定版本的。没有历史就拿不到版本，构建会坏掉。设成 `0` 可以拉全部历史。

### 3. 踩坑记录

写这篇文章其实就是为了记录这个…

从这里开始才是正题（？）。从第一次部署流水线到真正发布到 PyPI…老实说改了挺多次。整个过程都老老实实留在提交历史里了(..)

#### 3-1. numpy 构建依赖和 manylinux 容器

cibuildwheel 在 Linux 下是在叫 manylinux 的 Docker 容器里执行构建的。manylinux 是一个标准构建环境，用来保证"这个 wheel 能在大多数 Linux 上运行"。容器版本不同，glibc（Linux 核心 C 库）版本也不同，版本越低兼容的 Linux 越多，但可能装不上最新的包。

最初用的是默认的 manylinux1（非常老的环境）容器，结果因为没有 `numpy>=1.25.0` 的 wheel，构建失败了。

修起来很简单。明确指定 `manylinux_2_28`（glibc 2.28）容器就行。

```toml
[tool.cibuildwheel.linux]
manylinux-x86_64-image = "manylinux_2_28"
```

构建时把 numpy 提到 `>=2.0`…用 numpy 2.0 构建出来的 wheel 在 numpy 1.25.0+ 的运行时是向后兼容的。所以运行时依赖设成 `numpy>=1.25.0`，让用户不一定非要用 numpy 2.0。

numpy 2.0 出来挺久了，但…还没用上的项目还是更多…

#### 3-2. cibuildwheel v2 → v3

最开始用的是 `pypa/cibuildwheel@v2`，但…v2 标签消失了，构建失败。

升到 `@v3`，并为了可重现性 pin 到 `@v3.4.0`。

#### 3-3. 移除 macOS 支持

本来既有的包是支持 macOS 的，没必要把现有的去掉，所以连 `macos-13`（Intel）和 `macos-14`（Apple Silicon）都加上了…

…但 `macos-13` runner 在 GitHub Actions 上不再支持了。我做这个本来是为了让自己省事，结果在这种地方折腾有意义吗…于是直接把 macOS 整个去掉。从构建矩阵、pyproject.toml classifier、README 全部删掉 macOS 相关内容。

谁开 issue 的时候再看吧 👀

#### 3-4. PyPI 发布方式变更、TOML 转义等

除此之外还有些零碎的踩坑…

- PyPI 发布方式改了三次：**OIDC → API token → twine**
- TOML 的 single-quote 和 double-quote 转义处理差异搞坏了 setuptools_scm 模板
- 等等…

本以为很快就能搞定…还是再次感受到老项目构建是个吃时间的怪物。

### 4. 自动检测 Python 新版本 (`check_new_python.yml`)

其实做这个的原因…一开始这个项目的动机就是"Python 出了新版本但旧包跟不上"嘛。我不想重蹈覆辙。

每次 Python 出新版本都要手动更新构建目标的话…终究只要有人忘了，就又会回到同样的状况。

所以我加了一个自动检测的工作流：

1. 每月 1 日 00:00 UTC 自动运行（也支持手动运行）
2. 在 [endoflife.date API](https://endoflife.date/api/python.json) 查询当前还没 EOL 的 Python 3.x 版本列表
3. 解析 `pyproject.toml` 里 cibuildwheel build 配置里的当前构建目标版本
4. 检测到新版本就更新 `pyproject.toml` 并 **自动创建 PR**

比如 Python 3.14 发布的话，工作流会检测到，并创建一个把它更新成 `build = "cp310-* cp311-* cp312-* cp313-* cp314-*"` 的 PR。

所以以后呢？我幻想着以后只要 PR 一点就能支持新版本了。


## 最终结果

- **包名**：`lemon-pyopenjtalk-prebuilt`
- **安装**：`pip install lemon-pyopenjtalk-prebuilt`
- **API**：与原版 `pyopenjtalk` 100% 一致（模块名也还是 `pyopenjtalk`）
- **支持的 Python**：3.9, 3.10, 3.11, 3.12, 3.13
- **支持的平台**：Linux x86_64、Windows AMD64
- **构建工具**：不需要（提供 prebuilt wheel）
- **自动化**：tag push 时自动构建发布，每月自动检测 Python 新版本

按文件变更看：

```
 .github/workflows/build_and_release.yml |  73 ++++++++++++   (新增)
 .github/workflows/check_new_python.yml  | 108 +++++++++++++++++ (新增)
 .github/workflows/ci.yaml               | 127 -----------------  (删除)
 pyproject.toml                          |  78 +++++--------  (重构)
 README.md                               | 194 ++++++++-----------  (用韩语重写)
 README.en.md                            |  94 +++++++++++++++ (新增)
 README.ja.md                            |  94 +++++++++++++++ (新增)
 README.zh.md                            |  94 +++++++++++++++ (新增)
 10 files changed, 554 insertions(+), 312 deletions(-)
```

## 收尾

老实说，开始的时候我以为"改改 pyproject.toml 写个 GitHub Actions 不就完事了？"…但构建环境（manylinux 容器）、发布方式（OIDC/token/twine）、平台支持范围、版本管理自动化…再次感受到，估工时的时候要把第一感觉的两倍报上去。

不过…终归做完之后，就是一行 `pip install lemon-pyopenjtalk-prebuilt` 就能搞定的包了，Bert-VITS2 那边的依赖也能减少一个，所以挺满足。

包在 [PyPI](https://pypi.org/project/lemon-pyopenjtalk-prebuilt/)，源码在 [GitHub](https://github.com/LemonDouble/lemon-pyopenjtalk-prebuilt) 上可以找到！
