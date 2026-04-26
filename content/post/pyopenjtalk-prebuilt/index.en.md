---
title: "Forking a prebuilt wheel package to support newer Python versions because the upstream stopped updating (Pyopenjtalk)"
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

_Note: I'm based in Korea, so some context here is Korea-specific._

A while back I wrote a [post about building a TTS with Bert-VITS2](../make-vits2-tts/). I've been using that TTS pretty happily ever since. But… the Bert-VITS2 repo development just stopped.

When a repo stops getting updates, what's the problem? Bumping Python versions and managing dependencies becomes painful.

In my case, I have an "ML Server" on my home server where I've crammed all my ML-related stuff. So I've installed all sorts of things on it… and when there's a stale repo like that, conflicts keep popping up whenever I try to install something. For example, numpy 2.0…

So even when I just wanted to add one more thing… every install kept running into conflicts, which made it not so easy. Then I started digging — who exactly keeps breaking my dependencies? Turned out it was that package. The one that wouldn't let me bump past Python 3.10… I want to taste 3.12 too, you know.

So after enduring it for a while… I just decided to do it myself. Once I saw that **pyopenjtalk** was the reason I couldn't go past Python 3.10, I thought: oh, if I just fix that one thing, I'm done!

But this thing builds a C module into a wheel, so it was a bit tricky to touch. Still… anyway, I pulled it off.


## pyopenjtalk?

[pyopenjtalk](https://github.com/r9y9/pyopenjtalk) is a Python wrapper for [OpenJTalk](https://open-jtalk.sourceforge.net/), a Japanese TTS engine. It converts Japanese text into speech, or into phonetic symbols.

```python
import pyopenjtalk

# TTS: text → audio waveform
x, sr = pyopenjtalk.tts("おめでとうございます")

# G2P: text → pronunciation
pyopenjtalk.g2p("こんにちは")           # → 'k o N n i ch i w a'
pyopenjtalk.g2p("こんにちは", kana=True) # → 'コンニチワ'
```

Internally, OpenJTalk (written in C/C++) is bound via Cython, and the native code is built with CMake.

## The problem: installing it is too hard

pyopenjtalk only ships a **source distribution (sdist)** on PyPI. Running `pip install pyopenjtalk` means the user's machine has to compile C++ code directly. So before installing, you need all of this:

- CMake
- A C++ compiler (gcc, MSVC, etc.)
- Cython

You can somehow set this up on a local dev machine, but… on Docker images, CI environments, or servers without build tools, installation isn't easy.

To solve this, there used to be a package called [pyopenjtalk-prebuilt](https://pypi.org/project/pyopenjtalk-prebuilt/), which Bert-VITS2 was using. I think GPT-SOVITS used it too, and it pops up here and there in Japanese TTS projects.
It provided prebuilt wheels so you could install without build tools, but… **updates stopped at Python 3.11.** Python 3.12 or above? Can't use it.

The original pyopenjtalk hasn't been updated in over a year, prebuilt isn't getting uploaded either…

Well, if the code is frozen, can't I just fork it and use that? <<

So I ended up forking and building it myself.


## What I did

I forked the original pyopenjtalk repo and built a pipeline from scratch to build and publish prebuilt wheels. The package name is `lemon-pyopenjtalk-prebuilt`.

The work splits roughly into four parts:

1. Restructure the package config
2. Build the CI/CD pipeline
3. Stabilize the publish pipeline (yak shaving)
4. Add an automatic Python version detection workflow

### 1. Restructuring the package config (`pyproject.toml`)

I cleaned up the original pyopenjtalk's `pyproject.toml` to fit the new package.

- **Package name**: `pyopenjtalk` → `lemon-pyopenjtalk-prebuilt`
- **Minimum Python**: 3.8 → 3.9
- **numpy**: build with `numpy>=2.0`, but at runtime `numpy>=1.25.0` is OK (backward compatible)
- **Build target Python**: 3.8 ~ 3.10 → 3.9 ~ 3.13
- **Supported OS**: Linux, macOS, Windows → Linux (x86_64), Windows (AMD64)

And I dropped all the unused stuff…

- `docs`, `dev` optional-dependencies — Sphinx (doc generation), pysen (linter), mypy (type checker), etc. We only need to ship built wheels, so unnecessary
- The entire pysen lint config — code style settings from the original repo. No reason to maintain that in a fork. It's been frozen for over a year, so… it'll probably be fine
- Python 3.8 conditional dependencies — `importlib_resources` (resource access on 3.8), `oldest-supported-numpy` (numpy for 3.8), etc. Since I bumped minimum to 3.9, these aren't needed

And the core piece: I added the **[cibuildwheel](https://cibuildwheel.pypa.io/)** config. It's a tool maintained by PyPA — if you declare combinations like "CPython 3.9~3.13 × Linux/Windows" in `pyproject.toml`, GitHub Actions builds a wheel for each combination automatically. No need to manually install Python on each OS and run build scripts.

With this, a single workflow run produces **10 wheels + 1 sdist** (5 Linux + 5 Windows).

For versioning, I configured [setuptools_scm](https://github.com/pypa/setuptools-scm) to derive the version automatically from git tags.

### 2. Building the CI/CD pipeline (`build_and_release.yml`)

The original repo's CI was a source-build + test + sdist publish setup. I deleted that and wrote a new cibuildwheel-based wheel build + publish workflow.

```
Trigger: version tag (v*) push or manual run

┌─────────────────┐   ┌──────────────┐
│ build_wheels     │   │ build_sdist  │
│ (ubuntu, windows)│   │ (ubuntu)     │
│ cibuildwheel     │   │ python -m    │
│ → .whl artifacts │   │   build      │
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

[twine](https://twine.readthedocs.io/) is a CLI tool that uploads built wheels or sdists to PyPI. Run `twine upload dist/*` and your build artifacts go up to PyPI.

And for GitHub Actions checkout, two options are critical:

- **`submodules: recursive`** — pyopenjtalk has the OpenJTalk C++ source as a git submodule. Without this option, the submodule folder gets checked out empty, so there's no C++ source to build, and naturally the build fails.
- **`fetch-depth: 0`** — by default, GitHub Actions only fetches the latest commit (shallow clone), but setuptools_scm looks at git tag history to determine the version. Without history, it can't figure out the version and the build breaks. Setting it to `0` fetches the full history.

### 3. The yak shaving log

Honestly, I wrote this post mainly to record this part…

This is where it gets real. Between first putting up the pipeline and actually getting it published to PyPI… honestly, I revised it quite a few times. The whole journey is preserved in the commit history (..)

#### 3-1. numpy build dependency and the manylinux container

cibuildwheel builds inside a Docker container called manylinux on Linux. manylinux is the standard build environment that guarantees "this wheel runs on most Linux systems." Different container versions ship different glibc (Linux's core C library) versions — lower versions are compatible with more Linux distros, but newer packages might not install.

Initially I used the default manylinux1 container (a very old environment), and the build failed because no `numpy>=1.25.0` wheel exists for it.

The fix was simple. I just had to explicitly specify the `manylinux_2_28` (glibc 2.28) container.

```toml
[tool.cibuildwheel.linux]
manylinux-x86_64-image = "manylinux_2_28"
```

And I bumped the build-time numpy to `>=2.0`. Wheels built with numpy 2.0 are backward compatible at runtime down to numpy 1.25.0+. So I set the runtime dependency to `numpy>=1.25.0`, so users don't have to be on numpy 2.0.

Numpy 2.0 has been out for a while, but… there are still way more projects that haven't moved to it.

#### 3-2. cibuildwheel v2 → v3

I started with `pypa/cibuildwheel@v2`, but… the v2 tag was retired and the build failed.

I bumped it to `@v3` and pinned to `@v3.4.0` for reproducibility.

#### 3-3. Removing macOS support

The original package had macOS support, so I figured I'd keep it since there was no need to drop existing support, and added `macos-13` (Intel) and `macos-14` (Apple Silicon)…

…and the `macos-13` runner was discontinued on GitHub Actions. Is it really worth shaving this yak just to make my own life easier? I dropped macOS entirely. Removed all macOS-related stuff from the build matrix, the pyproject.toml classifier, and the README.

If someone opens an issue, I'll deal with it then 👀

#### 3-4. Changing PyPI publish method, TOML escaping, etc.

There were other small detours too…

- Switched the PyPI publish method **OIDC → API token → twine** three times
- The difference between TOML single-quote and double-quote escaping broke the setuptools_scm template
- Etc.

I thought I'd be done quickly, but… once again, I was reminded that building old projects is a time-eating monster.

### 4. Automatic Python version detection (`check_new_python.yml`)

Honestly, the reason I built this is… the whole motivation for this project was "a new Python version came out and the old package didn't follow." I didn't want to repeat the same problem.

Manually updating the build target every time a new Python version drops… eventually someone forgets and you're back in the same situation.

So I added a workflow that detects it automatically:

1. Runs automatically at 00:00 UTC on the 1st of each month (manual run also possible)
2. Queries the [endoflife.date API](https://endoflife.date/api/python.json) for the list of Python 3.x versions that are not currently EOL
3. Parses the current build target versions from the cibuildwheel build setting in `pyproject.toml`
4. If a new version is detected, updates `pyproject.toml` and **automatically opens a PR**

For example, when Python 3.14 is released, the workflow detects it and opens a PR updating to `build = "cp310-* cp311-* cp312-* cp313-* cp314-*"`.

So I imagined: from now on, I'll just click-merge the PR and support new versions.


## Final result

- **Package name**: `lemon-pyopenjtalk-prebuilt`
- **Install**: `pip install lemon-pyopenjtalk-prebuilt`
- **API**: 100% identical to the original `pyopenjtalk` (the module name is still `pyopenjtalk`)
- **Supported Python**: 3.9, 3.10, 3.11, 3.12, 3.13
- **Supported platforms**: Linux x86_64, Windows AMD64
- **Build tools**: not required (prebuilt wheels provided)
- **Automation**: auto build & publish on tag push, monthly auto-detection of new Python versions

In terms of file changes:

```
 .github/workflows/build_and_release.yml |  73 ++++++++++++   (new)
 .github/workflows/check_new_python.yml  | 108 +++++++++++++++++ (new)
 .github/workflows/ci.yaml               | 127 -----------------  (removed)
 pyproject.toml                          |  78 +++++--------  (restructured)
 README.md                               | 194 ++++++++-----------  (rewritten in Korean)
 README.en.md                            |  94 +++++++++++++++ (new)
 README.ja.md                            |  94 +++++++++++++++ (new)
 README.zh.md                            |  94 +++++++++++++++ (new)
 10 files changed, 554 insertions(+), 312 deletions(-)
```

## Wrapping up

Honestly, when I started I thought "isn't this just patching pyproject.toml and writing one GitHub Actions workflow?" But… the build environment (manylinux containers), publish methods (OIDC/token/twine), platform support range, version automation… once again I was reminded that when estimating effort, you should always double your initial guess.

But hey… in the end, after all that, it became a package you install with one line: `pip install lemon-pyopenjtalk-prebuilt`. And I got to drop one more dependency on the Bert-VITS2 side, so I'm satisfied.

You can find the package on [PyPI](https://pypi.org/project/lemon-pyopenjtalk-prebuilt/), and the source on [GitHub](https://github.com/LemonDouble/lemon-pyopenjtalk-prebuilt)!
