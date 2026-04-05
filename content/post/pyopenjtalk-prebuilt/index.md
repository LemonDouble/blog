---
title: "개인 서버 파이썬 버전 올리고 싶은데 업뎃이 멈춰서 prebuilt wheel 패키지 포크떠서 파이썬 추가 버전 지원하게 한 이야기  (Pyopenjtalk)"
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

예전에 [Bert-VITS2로 TTS 만드는 글](../make-vits2-tts/)을 올린 적이 있었는데요. 그때 만들어 둔 TTS를 꽤 잘 쓰고 있었습니다. 근데.. Bert-VITS2 레포가 개발이 멈춰버렸어요.

레포가 멈추면 뭐가 문제냐면.. 파이썬 버전 올리기가 힘들고 의존성 관리가 힘듭니다.

저같은 경우는, 홈서버에 ML Server라는데다가 ML 관련 기능을 다 짱박아뒀거든요. 그래서 뭐 깔아둔게 이것저것 많은데.. 저런 업데이트 안 되는 레포가 있으면 뭐 깔다가 자꾸 충돌이 나요. 예를 들어 numpy 2.0이라던가..

그래서 뭐 하나 추가하려고 하다가도.. 맨날 뭐 깔다가 충돌나니까 쉽지 않았습니다. 그래서 도대체 자꾸 의존성 꺠먹는게 누구야? 하니까 저 패키지더라구요. python 3.10 이상으로 못 올리게 하는 친구... 나도 3.12 맛좀 보고싶은데..

그래서 참다참다가.. 그냥 직접 하기로 했습니다. 파이썬 3.10 이상으로 못올리는게 **pyopenjtalk** 때문이란걸 보고!! 아 저것만 어떻게 하면 되곘구나! 한거죠.

근데 얘가 C 모듈 빌드해서 휠 만들어둔거라 건들기가 좀 어렵긴 하더라구요.. 그래도? 아무튼? 해냈습니다.

---

## pyopenjtalk?

[pyopenjtalk](https://github.com/r9y9/pyopenjtalk)은 일본어 TTS 엔진인 [OpenJTalk](https://open-jtalk.sourceforge.net/)의 Python 래퍼입니다. 일본어 텍스트를 음성으로 변환하거나, 발음 기호로 변환하는 기능을 제공해요.

```python
import pyopenjtalk

# TTS: 텍스트 → 음성 파형
x, sr = pyopenjtalk.tts("おめでとうございます")

# G2P: 텍스트 → 발음
pyopenjtalk.g2p("こんにちは")           # → 'k o N n i ch i w a'
pyopenjtalk.g2p("こんにちは", kana=True) # → 'コンニチワ'
```

내부적으로 C/C++로 작성된 OpenJTalk을 Cython으로 바인딩하고, CMake로 네이티브 코드를 빌드하는 구조예요.

## 문제: 설치가 너무 어렵다

pyopenjtalk은 PyPI에 **소스 배포판(sdist)만** 올라가 있습니다. `pip install pyopenjtalk`을 실행하면 사용자 환경에서 C++ 코드를 직접 컴파일해야 돼요. 그러니까 설치 전에 이게 다 있어야 합니다:

- CMake
- C++ 컴파일러 (gcc, MSVC 등)
- Cython

로컬 개발 환경이야 어떻게든 맞출 수 있지만.. Docker 이미지나 CI 환경, 빌드 도구 없는 서버에서는 설치가 쉽지 않습니다.

이 문제를 해결하려고 [pyopenjtalk-prebuilt](https://pypi.org/project/pyopenjtalk-prebuilt/)라는 패키지가 있었고.. Bert-VITS2에서는 요걸 가져다 쓰고 있어요. GPT-SOVITS에서도 그랬던 것 같고.. 일본어 TTS 관련 프로젝트에서 간간히 보입니다.
사전 빌드된 wheel을 제공해서 빌드 도구 없이 바로 설치할 수 있었는데.. **Python 3.11에서 업데이트가 멈췄습니다.** Python 3.12 이상? 못 씁니다.

pyopenjtalk 원본도 1년 넘게 업데이트가 없고, prebuilt도 안 올라가고.. 

어차피 코드 프리징된거면 걍 포크떠서 쓰면 되는거 아님? << 

그래서 결국 직접 포크해서 만들었습니다.

---

## 뭘 했나

원본 pyopenjtalk 저장소를 포크하고, prebuilt wheel을 빌드·배포하는 파이프라인을 처음부터 구축했어요. 패키지명은 `lemon-pyopenjtalk-prebuilt`.

작업은 크게 네 가지로 나뉩니다:

1. 패키지 설정 재구성
2. CI/CD 파이프라인 구축
3. 배포 파이프라인 안정화 (삽질)
4. 자동 Python 버전 감지 워크플로우 추가

### 1. 패키지 설정 재구성 (`pyproject.toml`)

원본 pyopenjtalk의 `pyproject.toml`을 새 패키지에 맞게 정리했습니다.

- **패키지명**: `pyopenjtalk` → `lemon-pyopenjtalk-prebuilt`
- **Python 최소 버전**: 3.8 → 3.9
- **numpy**: 빌드는 `numpy>=2.0`으로 하되, 실제 사용 시에는 `numpy>=1.25.0`이면 OK (하위 호환)
- **빌드 대상 Python**: 3.8 ~ 3.10 → 3.9 ~ 3.13
- **지원 OS**: Linux, macOS, Windows → Linux (x86_64), Windows (AMD64)

그리고 안 쓰는 파일들은 다 지우고... 

- `docs`, `dev` optional-dependencies — Sphinx(문서 생성), pysen(린터), mypy(타입 체크) 등 개발용 도구들. 우리는 빌드된 wheel만 배포하면 되니까 필요 없음
- pysen 린터 설정 전체 — 원본 레포의 코드 스타일 검사 설정. 굳이 포크에서 관리할 이유 없음. 업데이트 끊긴지 1년 넘었으니.. 잘 되겠지 뭐..
- Python 3.8 관련 조건부 의존성 — `importlib_resources`(3.8에서 리소스 접근용), `oldest-supported-numpy`(3.8용 numpy) 등. 최소 버전을 3.9로 올렸으니 불필요

그리고 핵심인 **[cibuildwheel](https://cibuildwheel.pypa.io/)** 설정을 추가했어요. PyPA에서 관리하는 도구인데, "CPython 3.9~3.13 × Linux/Windows" 같은 조합을 `pyproject.toml`에 선언해두면, GitHub Actions에서 알아서 각 조합마다 wheel을 빌드해줍니다. 직접 OS별로 Python 깔고 빌드 스크립트 돌릴 필요가 없어요.

이걸로 한 번의 워크플로우 실행에 **10개 wheel + 1개 sdist**가 만들어집니다. (Linux 5개 + Windows 5개)

버전 관리는 [setuptools_scm](https://github.com/pypa/setuptools-scm)으로, git 태그에서 자동으로 버전을 따오도록 설정했고요.

### 2. CI/CD 파이프라인 구축 (`build_and_release.yml`)

원본 저장소의 CI는 소스 빌드 + 테스트 + sdist 배포 구조였어요. 이걸 삭제하고, cibuildwheel 기반의 wheel 빌드 + 배포 워크플로우를 새로 작성했습니다.

```
트리거: 버전 태그 (v*) 푸시 또는 수동 실행

┌─────────────────┐   ┌──────────────┐
│ build_wheels     │   │ build_sdist  │
│ (ubuntu, windows)│   │ (ubuntu)     │
│ cibuildwheel     │   │ python -m    │
│ → .whl 아티팩트  │   │   build      │
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

[twine](https://twine.readthedocs.io/)은 빌드된 wheel이나 sdist를 PyPI에 업로드해주는 CLI 도구예요. `twine upload dist/*` 하면 빌드 결과물이 PyPI에 올라갑니다.

그리고 GitHub Actions에서 checkout 할 때 두 가지 옵션이 핵심인데요:

- **`submodules: recursive`** — pyopenjtalk은 내부에 OpenJTalk C++ 소스를 git submodule로 갖고 있어요. 이 옵션이 없으면 서브모듈 폴더가 빈 채로 체크아웃돼서, C++ 빌드할 소스가 없으니까 당연히 실패합니다.
- **`fetch-depth: 0`** — 기본적으로 GitHub Actions는 최신 커밋 1개만 가져오는데(shallow clone), setuptools_scm은 git 태그 히스토리를 보고 버전을 결정하거든요. 히스토리가 없으면 버전을 못 알아내서 빌드가 깨져요. `0`으로 설정하면 전체 히스토리를 가져옵니다.

### 3. 삽질의 기록

이거 기록하려고 글 썼는데.. 

여기서부터가 본론(?)입니다. 처음 파이프라인을 올리고 실제로 PyPI에 배포되기까지.. 솔직히 좀 많이 고쳤어요. 커밋 히스토리에 그 과정이 고스란히 남아있습니다(..)

#### 3-1. numpy 빌드 의존성과 manylinux 컨테이너

cibuildwheel은 Linux에서 manylinux라는 Docker 컨테이너 안에서 빌드를 수행하는데요. manylinux는 "이 wheel은 대부분의 Linux에서 돌아갑니다"를 보장하기 위한 표준 빌드 환경이에요. 컨테이너 버전마다 glibc(Linux의 핵심 C 라이브러리) 버전이 다른데, 낮은 버전일수록 더 많은 Linux에서 호환되지만 최신 패키지가 설치 안 될 수 있습니다.

처음에는 기본 manylinux1 (아주 오래된 환경) 컨테이너를 사용했더니, `numpy>=1.25.0`의 wheel이 존재하지 않아서 빌드가 실패했어요.

해결은 간단했어요. `manylinux_2_28` (glibc 2.28) 컨테이너를 명시적으로 지정하면 됐습니다.

```toml
[tool.cibuildwheel.linux]
manylinux-x86_64-image = "manylinux_2_28"
```

그리고 빌드 시 numpy를 `>=2.0`으로 올렸는데.. numpy 2.0으로 빌드한 wheel은 numpy 1.25.0+ 런타임에서 하위 호환되거든요. 그래서 런타임 의존성은 `numpy>=1.25.0`으로 설정해서, 사용자가 꼭 numpy 2.0을 안 써도 되게 했습니다.

numpy 2.0 나온지 한참 됐지만.. 아직 안 쓰는 프로젝트가 훨씬 많으니까요..

#### 3-2. cibuildwheel v2 → v3

처음에는 `pypa/cibuildwheel@v2`를 사용했는데.. v2 태그가 소멸되어 빌드가 실패. 

`@v3`으로 올리고, 재현성을 위해 `@v3.4.0`으로 핀 고정했습니다.

#### 3-3. macOS 지원 제거

원래 기존 패키지에 macOS 지원이 있어서.. 굳이 있는거 뺄 필요는 없길래 가져가려고 해서 `macos-13` (Intel)이랑 `macos-14` (Apple Silicon)까지 넣었는데..

`macos-13` 러너가 GitHub Actions에서 지원 종료. 나 편하자고 하는건데 이거 삽질하는데 의미가 있을까... 싶어서 macOS는 아예 빼버렸습니다. 빌드 매트릭스, pyproject.toml classifier, README 전부에서 macOS 관련 내용을 삭제.

누가 이슈열면 그때 보겠죠 👀

#### 3-4. PyPI 배포 방식 변경, TOML 이스케이프 등

이거 말고도 자잘한 삽질이 좀 있었는데요..

- PyPI 배포 방식을 **OIDC → API 토큰 → twine**으로 세 번 바꿈
- TOML의 single-quote와 double-quote 이스케이프 처리 차이로 setuptools_scm 템플릿이 깨짐
- 등등..

금방 할 줄 알았는데.. 역시 오래된 프로젝트 빌드는 시간 잡아먹는 괴물이란걸 새삼 다시 느꼈습니다.

### 4. 자동 Python 버전 감지 (`check_new_python.yml`)

사실 이걸 만든 이유가.. 애초에 이 프로젝트를 시작한 동기가 "Python 새 버전 나왔는데 기존 패키지가 안 따라와줘서" 잖아요. 그래서 같은 문제를 반복하고 싶지 않았습니다.

새로운 Python 버전이 나올 때마다 수동으로 빌드 대상을 업데이트하는 건.. 결국 또 누군가가 까먹으면 같은 상황이 되니까요.

그래서 자동으로 감지하는 워크플로우를 추가했어요:

1. 매월 1일 00:00 UTC에 자동 실행 (수동 실행도 가능)
2. [endoflife.date API](https://endoflife.date/api/python.json)에서 현재 EOL이 아닌 Python 3.x 버전 목록 조회
3. `pyproject.toml`의 cibuildwheel build 설정에서 현재 빌드 대상 버전 파싱
4. 새 버전이 감지되면 `pyproject.toml`을 업데이트하고 **자동으로 PR 생성**

예를 들어 Python 3.14가 릴리스되면, 워크플로우가 이를 감지하고 `build = "cp310-* cp311-* cp312-* cp313-* cp314-*"`로 업데이트하는 PR을 만들어줘요. 

저는 그래서? 앞으로는 그냥 PR 딸깍 하고 눌러주면 새 버전 지원하는 상상을 했습니다.

---

## 최종 결과

- **패키지명**: `lemon-pyopenjtalk-prebuilt`
- **설치**: `pip install lemon-pyopenjtalk-prebuilt`
- **API**: 원본 `pyopenjtalk`과 100% 동일 (모듈명도 `pyopenjtalk` 그대로)
- **지원 Python**: 3.9, 3.10, 3.11, 3.12, 3.13
- **지원 플랫폼**: Linux x86_64, Windows AMD64
- **빌드 도구**: 불필요 (prebuilt wheel 제공)
- **자동화**: 태그 푸시 시 자동 빌드·배포, 매월 새 Python 버전 자동 감지

파일 변경 기준으로 보면:

```
 .github/workflows/build_and_release.yml |  73 ++++++++++++   (신규)
 .github/workflows/check_new_python.yml  | 108 +++++++++++++++++ (신규)
 .github/workflows/ci.yaml               | 127 -----------------  (삭제)
 pyproject.toml                          |  78 +++++--------  (재구성)
 README.md                               | 194 ++++++++-----------  (한국어로 재작성)
 README.en.md                            |  94 +++++++++++++++ (신규)
 README.ja.md                            |  94 +++++++++++++++ (신규)
 README.zh.md                            |  94 +++++++++++++++ (신규)
 10 files changed, 554 insertions(+), 312 deletions(-)
```

## 마무리

솔직히 시작할 때는 "pyproject.toml 좀 고치고 GitHub Actions 하나 짜면 끝 아닌가?" 싶었는데.. 빌드 환경(manylinux 컨테이너), 배포 방식(OIDC/토큰/twine), 플랫폼 지원 범위, 버전 관리 자동화까지.. 역시 공수잡을땐 처음 생각했던거 두배는 불러야한다는 생각을 다시한번.. 했읍니다

근데 뭐.. 결국 하고 나니까 `pip install lemon-pyopenjtalk-prebuilt` 한 줄이면 끝나는 패키지가 됐고, Bert-VITS2 쪽 의존성도 하나 줄일 수 있게 됐으니 만족합니다.

패키지는 [PyPI](https://pypi.org/project/lemon-pyopenjtalk-prebuilt/)에서, 소스는 [GitHub](https://github.com/LemonDouble/lemon-pyopenjtalk-prebuilt)에서 확인할 수 있습니다!
