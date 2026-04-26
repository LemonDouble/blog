---
title: "Fly.ioに載せやすいサービス (Uptime Kuma)"
slug: 7da6be543113ac58
date: 2023-01-22T20:41:31+09:00
image: cover.png
math: 
categories:
    - SaaS
    - Fly.io
tags:
    - セルフホスティング
    - Uptime Kuma
    - モニタリング
    - Docker
    - Fly.io
---

_注：筆者は韓国在住のため、本文には韓国特有の文脈が含まれることがあります。_

今回紹介するプロジェクトは、サーバーのUptimeを記録してくれるUptime Kumaというサービスです。

自分のクラスタ/サーバーなどをモニタリングし、問題が発生した場合にはTelegram、Slack、Discord、E-mail...などのさまざまな方法でサーバーがダウンしたことを通知してくれます。

![](2023-01-22-20-50-49.png)

こんな感じです！

GitHubのページ ( [Link](https://github.com/louislam/uptime-kuma) ) に行けば、Demoサービスも体験できます。

Fly.ioについての紹介は以前の記事 ([Fly.ioの紹介とFly.ioに載せやすいサービスのおすすめ (VaultWarden)](/ja/p/fly.io-%EC%86%8C%EA%B0%9C-%EB%B0%8F-fly.io%EC%97%90-%EC%98%AC%EB%A6%AC%EA%B8%B0-%EC%A2%8B%EC%9D%80-%EC%84%9C%EB%B9%84%EC%8A%A4-%EC%B6%94%EC%B2%9C-vaultwarden/)) でたくさんしましたので、その記事を参考にしていただき、今回の記事ではインストール方法だけに集中します。

### 1. fly.tomlファイルの作成

* 適切な場所に移動し、`flyctl launch`を入力してfly.tomlファイルを作成します。

### 2. volumeの作成

* モニタリング記録の保存、ID/Passwordの保存などのためにVolumeが必要です。
* それほど多くのボリュームは必要ないので、私の場合は1GBで作成しました。

```bash
flyctl volume create uptime_kuma_data --region nrt --size 1
```


### 3. fly.tomlファイルの変更

* 次のtomlファイルを参考に、デプロイファイルを作成してください。
* 

```toml

# アプリ名はflyctl launch時に設定した値です。
app = "lemon-uptime-kuma"
kill_signal = "SIGINT"
kill_timeout = 5
processes = []

# 追加 :1 はDebian Stable Buildです。
# https://hub.docker.com/r/louislam/uptime-kuma
[build]
  image = "louislam/uptime-kuma:1"

[env]

# 追加、先ほど作成したVolumeをMountします。
[mounts]
source="uptime_kuma_data"
destination="/app/data"

[experimental]
  auto_rollback = true

[[services]]
  http_checks = []
  internal_port = 3001 # 変更、デフォルト設定ではKumaは内部で3001ポートを使用します。
  processes = ["app"]
  protocol = "tcp"
  script_checks = []
  [services.concurrency]
    hard_limit = 25
    soft_limit = 20
    type = "connections"

  [[services.ports]]
    force_https = true
    handlers = ["http"]
    port = 80

  [[services.ports]]
    handlers = ["tls", "http"]
    port = 443

  [[services.tcp_checks]]
    grace_period = "1s"
    interval = "15s"
    restart_limit = 0
    timeout = "2s"
```

### 4. デプロイ！

```
flyctl deploy 
```
コマンドを入力してデプロイします。

### 5. 接続してID/Passwordを設定する

`<アプリ名>.fly.dev`に接続するか、
アドレスがよくわからない場合は`https://fly.io/apps`に接続してアプリを選択し、アドレスを取得します。

その後、ID/Passwordを設定します。


### 6. モニタリングの追加

![](2023-01-22-21-01-00.png)

Add New Monitorをクリックしたあと、次のように設定します。

その後Saveボタンを押すと..

![](2023-01-22-21-01-56.png)

モニタリングが正常に動作します！

### Appendix 1. TMI

* 日本語もサポートしています！Settings -> Appearanceから言語設定ができます。（韓国語にも対応しています。）

### Appendix 2. Custom Domainを使う

* 私の場合、自分のドメインがあるのでIntegrationしようと思います。
* アプリページに行き、IPv6を取得します。（IPv4はShared IPv4なので、IPv6を使用しました。）

* 自分のDNSページに行き、`CNAME`タイプを選択し、値として`<アプリ名>.fly.dev`を入力します。
* その後、https://fly.io/apps ダッシュボードに入り、kumaアプリ -> Certificates -> Add Certificateを押した後、入力したカスタムドメインを入力します。

![](2023-01-22-21-32-09.png)

* しばらくすると、自分が作成したカスタムドメインで接続できることが確認できます！
