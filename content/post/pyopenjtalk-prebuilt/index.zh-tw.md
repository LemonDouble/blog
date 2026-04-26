---
title: "想升級個人伺服器的 Python 版本，但上游停止更新，於是 fork 了 prebuilt wheel 套件讓它支援新 Python 版本的故事 (Pyopenjtalk)"
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

_註：筆者居住於韓國，部分內容包含韓國特有的背景。_

之前我寫過一篇 [用 Bert-VITS2 做 TTS 的文章](../make-vits2-tts/)，那時候做的 TTS 一直用得挺好。但是…Bert-VITS2 的 repo 已經停止開發了。

repo 停止維護會有什麼問題呢？升級 Python 版本變得很麻煩，依賴管理也很麻煩。

我的情況是，在家用伺服器上有個叫 ML Server 的地方，把所有 ML 相關的功能全塞在那裡。所以裝了一堆雜七雜八的東西…如果有那種不更新的 repo 存在，每次裝東西都會衝突。例如 numpy 2.0 之類的…

所以哪怕只是想加一個東西…每次裝東西都衝突，搞得很不容易。我就開始追查「到底是誰一直在搞壞依賴？」，結果發現就是那個套件。不讓我升級到 Python 3.10 以上的傢伙…我也想嚐嚐 3.12 的味道啊。

於是忍了又忍…我決定自己動手。看到不讓我升級到 Python 3.10 以上的原因是 **pyopenjtalk** 之後，我心想：啊，只要把那個搞定就行了！

不過這傢伙是把 C 模組編譯成 wheel 的，要動它有點麻煩…但是？反正？我搞定了。


## pyopenjtalk?

[pyopenjtalk](https://github.com/r9y9/pyopenjtalk) 是日語 TTS 引擎 [OpenJTalk](https://open-jtalk.sourceforge.net/) 的 Python 封裝。它能把日語文字轉成語音，或者轉成發音符號。

```python
import pyopenjtalk

# TTS: 文字 → 語音波形
x, sr = pyopenjtalk.tts("おめでとうございます")

# G2P: 文字 → 發音
pyopenjtalk.g2p("こんにちは")           # → 'k o N n i ch i w a'
pyopenjtalk.g2p("こんにちは", kana=True) # → 'コンニチワ'
```

內部用 Cython 綁定了 C/C++ 寫成的 OpenJTalk，並用 CMake 建構原生程式碼。

## 問題：安裝太難了

pyopenjtalk 在 PyPI 上 **只發布了原始碼分發版（sdist）**。執行 `pip install pyopenjtalk` 時，需要在使用者環境直接編譯 C++ 程式碼。所以安裝前必須備齊這些：

- CMake
- C++ 編譯器（gcc、MSVC 等）
- Cython

本地開發環境還能湊合，但…在 Docker 映像、CI 環境或沒有建構工具的伺服器上，安裝就不容易了。

為了解決這個問題，曾經有個叫 [pyopenjtalk-prebuilt](https://pypi.org/project/pyopenjtalk-prebuilt/) 的套件，Bert-VITS2 就是用的它。我記得 GPT-SOVITS 也是…在日語 TTS 相關專案中時不時能看到。
它提供預先建構好的 wheel，可以不用建構工具直接安裝，但是…**在 Python 3.11 停止更新了。** Python 3.12 以上？用不了。

pyopenjtalk 原版超過一年沒更新，prebuilt 也不發新的…

反正程式碼已經凍結了，那直接 fork 來用不就行了？<<

於是最後我直接 fork 自己做了一個。


## 做了什麼

我 fork 了原版 pyopenjtalk repo，從零搭建了建構並發布 prebuilt wheel 的 pipeline。套件名叫 `lemon-pyopenjtalk-prebuilt`。

工作大致分四塊：

1. 重新整理套件設定
2. 建立 CI/CD pipeline
3. 穩定發布 pipeline（踩坑）
4. 新增自動偵測 Python 新版本的 workflow

### 1. 重新整理套件設定 (`pyproject.toml`)

我把原版 pyopenjtalk 的 `pyproject.toml` 整理成符合新套件的樣子。

- **套件名**：`pyopenjtalk` → `lemon-pyopenjtalk-prebuilt`
- **Python 最低版本**：3.8 → 3.9
- **numpy**：建構用 `numpy>=2.0`，但實際使用時 `numpy>=1.25.0` 就行（向後相容）
- **建構目標 Python**：3.8 ~ 3.10 → 3.9 ~ 3.13
- **支援的 OS**：Linux、macOS、Windows → Linux (x86_64)、Windows (AMD64)

然後把不用的檔案全刪了…

- `docs`、`dev` optional-dependencies — Sphinx（產生文件）、pysen（linter）、mypy（型別檢查）等開發工具。我們只需要發布建構好的 wheel，不需要這些
- 整個 pysen linter 設定 — 原 repo 的程式碼風格檢查設定。fork 沒必要管這個。都停更一年多了…應該沒事吧
- Python 3.8 相關的條件依賴 — `importlib_resources`（3.8 用來存取資源）、`oldest-supported-numpy`（3.8 用的 numpy）等。最低版本提到 3.9 後就用不上了

然後核心的 **[cibuildwheel](https://cibuildwheel.pypa.io/)** 設定加上了。這是 PyPA 維護的工具，在 `pyproject.toml` 裡宣告類似 「CPython 3.9~3.13 × Linux/Windows」 這樣的組合，GitHub Actions 就會為每個組合自動建構 wheel。不用自己在每個 OS 上裝 Python 跑建構腳本。

這樣一次 workflow 執行就能產出 **10 個 wheel + 1 個 sdist**（Linux 5 個 + Windows 5 個）。

版本管理用 [setuptools_scm](https://github.com/pypa/setuptools-scm)，設定成自動從 git tag 取版本號。

### 2. 建立 CI/CD pipeline (`build_and_release.yml`)

原 repo 的 CI 是原始碼建構 + 測試 + sdist 發布的結構。把這個刪掉，重新寫了基於 cibuildwheel 的 wheel 建構 + 發布 workflow。

```
觸發：版本 tag (v*) push 或手動執行

┌─────────────────┐   ┌──────────────┐
│ build_wheels     │   │ build_sdist  │
│ (ubuntu, windows)│   │ (ubuntu)     │
│ cibuildwheel     │   │ python -m    │
│ → .whl 產物      │   │   build      │
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

[twine](https://twine.readthedocs.io/) 是把建構好的 wheel 或 sdist 上傳到 PyPI 的 CLI 工具。執行 `twine upload dist/*` 就能把建構產物上傳到 PyPI。

GitHub Actions 在 checkout 時，有兩個選項很關鍵：

- **`submodules: recursive`** — pyopenjtalk 內部把 OpenJTalk C++ 原始碼作為 git submodule。沒有這個選項的話子模組資料夾會被空著 checkout，沒有 C++ 原始碼可以建構，理所當然會失敗。
- **`fetch-depth: 0`** — 預設情況下 GitHub Actions 只拉最新一個 commit（shallow clone），但 setuptools_scm 是看 git tag 歷史來決定版本的。沒有歷史就拿不到版本，建構會壞掉。設成 `0` 可以拉全部歷史。

### 3. 踩坑記錄

寫這篇文章其實就是為了記錄這個…

從這裡開始才是正題（？）。從第一次部署 pipeline 到真正發布到 PyPI…老實說改了挺多次。整個過程都老老實實留在 commit 歷史裡了(..)

#### 3-1. numpy 建構依賴和 manylinux 容器

cibuildwheel 在 Linux 下是在叫 manylinux 的 Docker 容器裡執行建構的。manylinux 是一個標準建構環境，用來保證「這個 wheel 能在大多數 Linux 上執行」。容器版本不同，glibc（Linux 核心 C 函式庫）版本也不同，版本越低相容的 Linux 越多，但可能裝不上最新的套件。

最初用的是預設的 manylinux1（非常老的環境）容器，結果因為沒有 `numpy>=1.25.0` 的 wheel，建構失敗了。

修起來很簡單。明確指定 `manylinux_2_28`（glibc 2.28）容器就行。

```toml
[tool.cibuildwheel.linux]
manylinux-x86_64-image = "manylinux_2_28"
```

建構時把 numpy 提到 `>=2.0`…用 numpy 2.0 建構出來的 wheel 在 numpy 1.25.0+ 的執行階段是向後相容的。所以執行階段依賴設成 `numpy>=1.25.0`，讓使用者不一定非要用 numpy 2.0。

numpy 2.0 出來挺久了，但…還沒用上的專案還是更多…

#### 3-2. cibuildwheel v2 → v3

最開始用的是 `pypa/cibuildwheel@v2`，但…v2 tag 消失了，建構失敗。

升到 `@v3`，並為了可重現性 pin 到 `@v3.4.0`。

#### 3-3. 移除 macOS 支援

本來既有的套件是支援 macOS 的，沒必要把現有的拿掉，所以連 `macos-13`（Intel）和 `macos-14`（Apple Silicon）都加上了…

…但 `macos-13` runner 在 GitHub Actions 上不再支援了。我做這個本來是為了讓自己省事，結果在這種地方折騰有意義嗎…於是直接把 macOS 整個拿掉。從建構矩陣、pyproject.toml classifier、README 全部刪掉 macOS 相關內容。

誰開 issue 的時候再看吧 👀

#### 3-4. PyPI 發布方式變更、TOML 跳脫等

除此之外還有些零碎的踩坑…

- PyPI 發布方式改了三次：**OIDC → API token → twine**
- TOML 的 single-quote 和 double-quote 跳脫處理差異搞壞了 setuptools_scm 範本
- 等等…

本以為很快就能搞定…還是再次感受到老專案建構是個吃時間的怪物。

### 4. 自動偵測 Python 新版本 (`check_new_python.yml`)

其實做這個的原因…一開始這個專案的動機就是「Python 出了新版本但舊套件跟不上」嘛。我不想重蹈覆轍。

每次 Python 出新版本都要手動更新建構目標的話…終究只要有人忘了，就又會回到同樣的狀況。

所以我加了一個自動偵測的 workflow：

1. 每月 1 日 00:00 UTC 自動執行（也支援手動執行）
2. 在 [endoflife.date API](https://endoflife.date/api/python.json) 查詢目前還沒 EOL 的 Python 3.x 版本列表
3. 解析 `pyproject.toml` 裡 cibuildwheel build 設定裡的目前建構目標版本
4. 偵測到新版本就更新 `pyproject.toml` 並 **自動建立 PR**

例如 Python 3.14 發布的話，workflow 會偵測到，並建立一個把它更新成 `build = "cp310-* cp311-* cp312-* cp313-* cp314-*"` 的 PR。

所以以後呢？我幻想著以後只要 PR 點一下就能支援新版本了。


## 最終結果

- **套件名**：`lemon-pyopenjtalk-prebuilt`
- **安裝**：`pip install lemon-pyopenjtalk-prebuilt`
- **API**：與原版 `pyopenjtalk` 100% 一致（模組名也還是 `pyopenjtalk`）
- **支援的 Python**：3.9, 3.10, 3.11, 3.12, 3.13
- **支援的平台**：Linux x86_64、Windows AMD64
- **建構工具**：不需要（提供 prebuilt wheel）
- **自動化**：tag push 時自動建構發布，每月自動偵測 Python 新版本

按檔案變更看：

```
 .github/workflows/build_and_release.yml |  73 ++++++++++++   (新增)
 .github/workflows/check_new_python.yml  | 108 +++++++++++++++++ (新增)
 .github/workflows/ci.yaml               | 127 -----------------  (刪除)
 pyproject.toml                          |  78 +++++--------  (重構)
 README.md                               | 194 ++++++++-----------  (用韓語重寫)
 README.en.md                            |  94 +++++++++++++++ (新增)
 README.ja.md                            |  94 +++++++++++++++ (新增)
 README.zh.md                            |  94 +++++++++++++++ (新增)
 10 files changed, 554 insertions(+), 312 deletions(-)
```

## 收尾

老實說，開始的時候我以為「改改 pyproject.toml 寫個 GitHub Actions 不就完事了？」…但建構環境（manylinux 容器）、發布方式（OIDC/token/twine）、平台支援範圍、版本管理自動化…再次感受到，估工時的時候要把第一感覺的兩倍報上去。

不過…終歸做完之後，就是一行 `pip install lemon-pyopenjtalk-prebuilt` 就能搞定的套件了，Bert-VITS2 那邊的依賴也能減少一個，所以挺滿足。

套件在 [PyPI](https://pypi.org/project/lemon-pyopenjtalk-prebuilt/)，原始碼在 [GitHub](https://github.com/LemonDouble/lemon-pyopenjtalk-prebuilt) 上可以找到！
