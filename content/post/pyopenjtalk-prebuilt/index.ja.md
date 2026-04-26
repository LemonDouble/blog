---
title: "個人サーバーのPythonバージョンを上げたかったのに更新が止まったprebuilt wheelパッケージをフォークしてPython追加バージョンに対応させた話 (Pyopenjtalk)"
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

_注：筆者は韓国在住のため、本文には韓国特有の文脈が含まれることがあります。_

以前 [Bert-VITS2でTTSを作る記事](../make-vits2-tts/) を書いたことがあるのですが、そのとき作っておいたTTSをかなり便利に使っていました。ところが…Bert-VITS2のリポジトリの開発が止まってしまいました。

リポジトリが止まると何が問題かというと…Pythonバージョンを上げるのが大変で、依存関係の管理も大変になります。

私の場合、ホームサーバーに「ML Server」というところにML関連の機能を全部詰め込んでいまして。なのでいろいろなものをインストールしているのですが…ああいう更新されないリポジトリがあると、何かをインストールするたびにコンフリクトが起きるんですね。例えばnumpy 2.0とか…

そういうわけで、何か一つ追加しようとしても…毎回インストール時に衝突するので、簡単にはいきませんでした。それで「いったい依存関係を壊し続けているのは誰なんだ？」と探っていったら、あのパッケージでした。Python 3.10以上に上げさせてくれない子…私も3.12を味わってみたいんですけど。

それで我慢に我慢を重ねた末…自分でやることにしました。Python 3.10以上に上げられない原因が **pyopenjtalk** だと分かって、「あ、あれだけ何とかすれば解決だ！」となったわけです。

ただ、これCモジュールをビルドしてwheelにしている関係で、いじるのが少し難しかったんですよ…。それでも？とにかく？やり遂げました。


## pyopenjtalk?

[pyopenjtalk](https://github.com/r9y9/pyopenjtalk) は日本語TTSエンジンである [OpenJTalk](https://open-jtalk.sourceforge.net/) のPythonラッパーです。日本語テキストを音声に変換したり、発音記号に変換する機能を提供してくれます。

```python
import pyopenjtalk

# TTS: テキスト → 音声波形
x, sr = pyopenjtalk.tts("おめでとうございます")

# G2P: テキスト → 発音
pyopenjtalk.g2p("こんにちは")           # → 'k o N n i ch i w a'
pyopenjtalk.g2p("こんにちは", kana=True) # → 'コンニチワ'
```

内部的にはC/C++で書かれたOpenJTalkをCythonでバインディングし、CMakeでネイティブコードをビルドする構成になっています。

## 問題：インストールが難しすぎる

pyopenjtalkはPyPIに **ソース配布版（sdist）のみ** が公開されています。`pip install pyopenjtalk` を実行すると、ユーザー環境でC++コードを直接コンパイルする必要があるんです。つまりインストール前にこれらが揃っていなければなりません：

- CMake
- C++コンパイラ（gcc、MSVCなど）
- Cython

ローカル開発環境ならどうにか揃えられますが…Dockerイメージや CI環境、ビルドツールが入っていないサーバーでは、インストールが簡単ではありません。

この問題を解決するために [pyopenjtalk-prebuilt](https://pypi.org/project/pyopenjtalk-prebuilt/) というパッケージが存在し、Bert-VITS2ではこれを使っていました。GPT-SOVITSでもそうだった気がしますし…日本語TTS関連プロジェクトでちらほら見かけます。
事前ビルドされたwheelを提供してくれていたので、ビルドツールなしですぐインストールできたのですが… **Python 3.11で更新が止まりました。** Python 3.12以上？使えません。

pyopenjtalk本家も1年以上更新がなく、prebuiltも上がってこないし…

どうせコードがフリーズしているなら、フォークして使えばいいんじゃない？ <<

ということで、結局自分でフォークして作りました。


## 何をやったか

オリジナルのpyopenjtalkリポジトリをフォークし、prebuilt wheelをビルド・公開するパイプラインを一から構築しました。パッケージ名は `lemon-pyopenjtalk-prebuilt` です。

作業は大きく4つに分かれます：

1. パッケージ設定の再構成
2. CI/CDパイプラインの構築
3. 公開パイプラインの安定化（試行錯誤）
4. Pythonバージョン自動検知ワークフローの追加

### 1. パッケージ設定の再構成 (`pyproject.toml`)

オリジナルpyopenjtalkの `pyproject.toml` を新パッケージ用に整理しました。

- **パッケージ名**：`pyopenjtalk` → `lemon-pyopenjtalk-prebuilt`
- **Python最小バージョン**：3.8 → 3.9
- **numpy**：ビルドは `numpy>=2.0` で行いつつ、実際の使用時は `numpy>=1.25.0` でOK（後方互換）
- **ビルド対象Python**：3.8 ～ 3.10 → 3.9 ～ 3.13
- **対応OS**：Linux、macOS、Windows → Linux (x86_64)、Windows (AMD64)

そして使っていないファイルは全部削除して…

- `docs`、`dev` optional-dependencies — Sphinx（ドキュメント生成）、pysen（リンター）、mypy（型チェッカー）など開発用ツール群。ビルド済みwheelだけ配布できればいいので不要
- pysenリンターの設定全体 — 元リポジトリのコードスタイル検査設定。フォークでわざわざ管理する理由なし。1年以上更新が止まっているので…まあ大丈夫でしょう
- Python 3.8関連の条件付き依存 — `importlib_resources`（3.8でのリソースアクセス用）、`oldest-supported-numpy`（3.8用numpy）など。最小バージョンを3.9に上げたので不要

そして核心となる **[cibuildwheel](https://cibuildwheel.pypa.io/)** の設定を追加しました。PyPAが管理しているツールで、「CPython 3.9～3.13 × Linux/Windows」のような組み合わせを `pyproject.toml` に宣言しておけば、GitHub Actionsが各組み合わせごとにwheelを自動でビルドしてくれます。OSごとにPythonをインストールしてビルドスクリプトを動かす必要がありません。

これで一回のワークフロー実行で **wheel 10個 + sdist 1個** が作られます（Linux 5個 + Windows 5個）。

バージョン管理は [setuptools_scm](https://github.com/pypa/setuptools-scm) で、gitタグから自動的にバージョンを取得するように設定しました。

### 2. CI/CDパイプラインの構築 (`build_and_release.yml`)

オリジナルリポジトリのCIはソースビルド + テスト + sdist公開という構成でした。これを削除して、cibuildwheelベースのwheelビルド + 公開ワークフローを新しく書き直しました。

```
トリガー：バージョンタグ (v*) のpush または手動実行

┌─────────────────┐   ┌──────────────┐
│ build_wheels     │   │ build_sdist  │
│ (ubuntu, windows)│   │ (ubuntu)     │
│ cibuildwheel     │   │ python -m    │
│ → .whl アーティファクト │ │   build      │
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

[twine](https://twine.readthedocs.io/) はビルド済みwheelやsdistをPyPIにアップロードしてくれるCLIツールです。`twine upload dist/*` を実行するとビルド成果物がPyPIにアップロードされます。

そして GitHub Actionsでcheckoutする際、2つのオプションが重要なのですが：

- **`submodules: recursive`** — pyopenjtalkは内部にOpenJTalkのC++ソースをgit submoduleとして持っています。このオプションがないとサブモジュールフォルダが空のままチェックアウトされ、C++をビルドするソースがないので当然失敗します。
- **`fetch-depth: 0`** — デフォルトだとGitHub Actionsは最新コミット1つだけ取得しますが（shallow clone）、setuptools_scmはgitタグの履歴を見てバージョンを決めます。履歴がないとバージョンを決定できずビルドが壊れます。`0` に設定すると全履歴を取得します。

### 3. 試行錯誤の記録

これを記録するためにこの記事を書いたんですが…

ここからが本題（？）です。最初にパイプラインを上げて実際にPyPIに公開されるまで…正直かなり修正しました。コミット履歴にその過程がまるまる残っています(..)

#### 3-1. numpyのビルド依存と manylinuxコンテナ

cibuildwheelはLinuxではmanylinuxというDockerコンテナの中でビルドを行います。manylinuxは「このwheelはほとんどのLinuxで動作します」を保証するための標準ビルド環境です。コンテナのバージョンによってglibc（Linuxの核となるCライブラリ）のバージョンが異なり、低いバージョンほど多くのLinuxで互換性があるものの、最新パッケージがインストールできないことがあります。

最初はデフォルトのmanylinux1（非常に古い環境）コンテナを使っていたのですが、`numpy>=1.25.0` のwheelが存在しなくてビルドが失敗しました。

解決は簡単でした。`manylinux_2_28`（glibc 2.28）コンテナを明示的に指定すればOKでした。

```toml
[tool.cibuildwheel.linux]
manylinux-x86_64-image = "manylinux_2_28"
```

そしてビルド時のnumpyを `>=2.0` に上げたのですが…numpy 2.0でビルドしたwheelはnumpy 1.25.0+のランタイムで後方互換が効くんですよ。なのでランタイム依存は `numpy>=1.25.0` に設定して、ユーザーが必ずしもnumpy 2.0を使わなくてもいいようにしました。

numpy 2.0が出てからかなり経ちますが…まだ使っていないプロジェクトの方がはるかに多いので…

#### 3-2. cibuildwheel v2 → v3

最初は `pypa/cibuildwheel@v2` を使っていたのですが…v2タグが消滅してビルドが失敗。

`@v3` に上げて、再現性のために `@v3.4.0` でピン留めしました。

#### 3-3. macOSサポートを削除

もともと既存パッケージにmacOSサポートがあったので…わざわざあるものを抜く必要もないかと思って `macos-13`（Intel）と `macos-14`（Apple Silicon）まで入れたのですが…

`macos-13` ランナーがGitHub Actionsでサポート終了。自分が楽になるためにやってるのに、こんなところで沼にハマる意味あるかな…と思って、macOSはきっぱり外しました。ビルドマトリクス、pyproject.tomlのclassifier、README全てからmacOS関連の内容を削除。

誰かissueを開いたらその時に見ます 👀

#### 3-4. PyPI公開方式の変更、TOMLエスケープなど

これ以外にも細々とした試行錯誤がありまして…

- PyPI公開方式を **OIDC → APIトークン → twine** に3回変更
- TOMLのsingle-quoteとdouble-quoteのエスケープ処理の違いでsetuptools_scmテンプレートが壊れた
- などなど…

すぐ終わると思っていたのに…やはり古いプロジェクトのビルドは時間を食う怪物だなと改めて感じました。

### 4. Pythonバージョン自動検知 (`check_new_python.yml`)

実はこれを作った理由が…そもそもこのプロジェクトを始めた動機が「Pythonの新バージョンが出たのに既存パッケージが追いついてくれない」じゃないですか。なので同じ問題を繰り返したくありませんでした。

新しいPythonバージョンが出るたびに手動でビルド対象を更新するのは…結局誰かが忘れたら同じ状況になってしまうので。

そこで自動で検知するワークフローを追加しました：

1. 毎月1日 00:00 UTCに自動実行（手動実行も可）
2. [endoflife.date API](https://endoflife.date/api/python.json) で現在EOLでないPython 3.xバージョンの一覧を取得
3. `pyproject.toml` のcibuildwheel build設定から、現在のビルド対象バージョンをパース
4. 新バージョンを検知したら `pyproject.toml` を更新して **自動でPRを作成**

例えばPython 3.14がリリースされたら、ワークフローがそれを検知して `build = "cp310-* cp311-* cp312-* cp313-* cp314-*"` に更新するPRを作ってくれます。

なので私はこれから？PRをポチっと押すだけで新バージョン対応する未来を想像しました。


## 最終結果

- **パッケージ名**：`lemon-pyopenjtalk-prebuilt`
- **インストール**：`pip install lemon-pyopenjtalk-prebuilt`
- **API**：オリジナル `pyopenjtalk` と100%同一（モジュール名も `pyopenjtalk` のまま）
- **対応Python**：3.9, 3.10, 3.11, 3.12, 3.13
- **対応プラットフォーム**：Linux x86_64、Windows AMD64
- **ビルドツール**：不要（prebuilt wheel提供）
- **自動化**：タグpush時に自動ビルド・公開、毎月新Pythonバージョンを自動検知

ファイル変更ベースで見ると：

```
 .github/workflows/build_and_release.yml |  73 ++++++++++++   (新規)
 .github/workflows/check_new_python.yml  | 108 +++++++++++++++++ (新規)
 .github/workflows/ci.yaml               | 127 -----------------  (削除)
 pyproject.toml                          |  78 +++++--------  (再構成)
 README.md                               | 194 ++++++++-----------  (韓国語で再作成)
 README.en.md                            |  94 +++++++++++++++ (新規)
 README.ja.md                            |  94 +++++++++++++++ (新規)
 README.zh.md                            |  94 +++++++++++++++ (新規)
 10 files changed, 554 insertions(+), 312 deletions(-)
```

## まとめ

正直、始めるときは「pyproject.toml をちょっと直してGitHub Actionsを一つ書けば終わりじゃない？」と思っていたのですが…ビルド環境（manylinuxコンテナ）、公開方式（OIDC/トークン/twine）、対応プラットフォームの範囲、バージョン管理の自動化まで…やはり工数を見積もるときは最初に思ったものの2倍は伝えなければならない、と改めて…思いました。

でもまあ…結局やり終えると `pip install lemon-pyopenjtalk-prebuilt` 一行で終わるパッケージになりましたし、Bert-VITS2側の依存も一つ減らせるようになったので満足しています。

パッケージは [PyPI](https://pypi.org/project/lemon-pyopenjtalk-prebuilt/) で、ソースは [GitHub](https://github.com/LemonDouble/lemon-pyopenjtalk-prebuilt) で確認できます！
