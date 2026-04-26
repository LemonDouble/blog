---
title: "Fly.ioの紹介とFly.ioにアップロードするのに適したサービスのおすすめ（VaultWarden）"
slug: 740c42a214d8ce6b
date: 2023-01-17T14:03:50Z
image: cover.png
categories:
    - SaaS
    - Fly.io
tags:
    - セルフホスティング
    - Vaultwarden
    - Bitwarden
    - パスワード管理
    - Docker
    - Fly.io
---

_注：筆者は韓国在住のため、本文には韓国特有の文脈が含まれることがあります。_

Fly.ioという会社は、Dockerイメージさえ用意すれば、簡単にDockerイメージを特定の国などにデプロイできる機能を提供しています。

（登録完了後）`fly.toml`というファイルを適切に作成し、

```bash
flyctl deploy
```

と入力するだけで、よろしくやってくれてイメージをサーバーにアップロードしてくれ、DNS、HTTPS設定、IP発行、モニタリング（Grafana）設定などをやってくれます。

さらに、数クリックでScale-upも行えます。自分でセットアップしてもよいのですが、結構面倒な作業を多く肩代わりしてくれて便利です。

また、フリーティアでも以下のようなスペックが提供されます。

![](image-9.png)


とはいえフリーティアなので、スペックはそれほど余裕があるわけではありません。
そのため、「データ管理が重要なのでクラウドサービスを利用したいが、それほど多くのコンピューティングリソースを使わない」アプリケーションの利用が向いています。

そういった用途にぴったりのパスワードマネージャー、Vaultwardenをインストールする方法を紹介します。


### VaultWarden（Bitwarden互換サーバー）

![](image-10.png)

Bitwardenはパスワードマネージャーで、Webブラウザ、Windows、iOS、Androidアプリなどを通じて、複数のデバイスでパスワードを共有できるようにしてくれます。

Chromeをお使いの方は、「Chromeの自動入力」機能をイメージしていただくと分かりやすいです。

Bitwardenはそれに加えてOTP機能をサポートしており、

![赤いボタンを押すと、Google OTPアプリを開かなくても認証できます！](image-12.png)

パスワードジェネレーターや、

![各サイトの設定に合わせて適切なパスワードを設定できます。](image-13.png)

Organization（組織）設定によるパスワード共有機能なども提供します。

一緒に開発をしている際に、Slack Webhook URLや環境変数、開発DBのIDおよびパスワードなどを共有する必要があるときに便利です。

それ以外にも、漏洩したDBの中に自分のパスワードと一致するものがあるか、再利用されているパスワードがあるかなどのセキュリティ監査機能や、エンドツーエンド暗号化を利用した機微なデータの送信機能であるSEND機能なども提供します。

ただ、Bitwardenはオープンソースなので、パスワードサーバーをSelf-hostingできるものの、プロダクションレベルで高可用性を確保するためには高いスペックと多くの設定が要求されます。（[Bitwarden Install Docs](https://bitwarden.com/help/install-on-premise-linux/)）

![256MB RAMで動かすにはスペックが…](image-14.png)


こうした問題を解決するため、Bitwarden Clientと互換性がありながらスペックを抑え、DatabaseもSQLiteを使うことで、Docker Imageひとつだけでもサーバーを起動できる[Vaultwarden](https://github.com/dani-garcia/vaultwarden)というプロジェクトがあります。


公式サーバーが2GB RAMを要求するのに対し、こちらのサーバーはIdle状態のときに約100MBのメモリしか使わないので、Fly.ioを利用してデプロイすれば手軽に利用できます！

![Vaultwardenデプロイ後にGrafanaで確認したRAM使用量](image-14.png)


次のステップに従って、Vaultwardenサーバーをデプロイしてみましょう。
1. Fly.io Docs（[Link](https://fly.io/docs/hands-on/install-flyctl/)）に従って、ご自分のOSに合ったflyctlをインストールし、ログインまで進めます。
2. シェルで任意のフォルダを1つ作成し（私の場合は`~/flyio/vaultwarden`というフォルダで作業しました）、そのフォルダへ移動します。
3 . 以下のコマンドを入力して`fly.toml`という設定ファイルを作成します。このとき、`app-name`はお好みに合わせて修正してください。入力時にRegionを選択するウィンドウが出るので、私は最も近いTokyo(nrt)リージョンを選択しました。

```bash
flyctl launch --name app-name --image vaultwarden/server:latest --no-deploy
```

4. 以下のコマンドを入力して、データを永続的に保存するためのVolumeを作成します。Volumeは、コンテナが落ちてもファイルが消えないように保存できるSSDのようなものとお考えください。アプリ名は3で設定したapp名と合わせ、Regionも同じく最も近いTokyo(nrt)リージョンを選びましょう。

```bash
flyctl volumes create vaultwarden_data --size 1 --app app-name
```

5. 以下のコマンドを使って、アプリケーション起動時に注入されるAdmin Token Secretsを作成します。`ADMIN_TOKEN`は管理者ページに入るためのパスワードのようなものとお考えください。
漏洩しないよう別途のパスワードを作成し、初期セットアップのためadminページに入れるようこのSecrets値を追加します。

```bash
flyctl secrets set ADMIN_TOKEN=abc123456 
```


6. フォルダにある`fly.toml`ファイルを、以下のファイルを参考に修正します。
既存ファイルから`[env]`部分、`[mounts]`部分、`[[services]]`->`internal_port`部分を修正すればOKです。

```yml
app = "app-name"
kill_signal = "SIGINT"
kill_timeout = 5
processes = []

[env]
  ROCKET_PORT = 8080

[experimental]
  allowed_public_ports = []
  auto_rollback = true

[mounts]
  destination = "/data"
  source = "vaultwarden_data"

[[services]]
  http_checks = []
  internal_port = 8080
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


7. 該当フォルダで以下のコマンドを実行してアプリをデプロイします。

```bash
flyctl deploy
```

8. デプロイ後、以下のコマンドを入力してWebページにアクセスします。その後、アカウント作成を押してアカウントの作成とマスターパスワードの作成を進めます。

```bash
flyctl open
```

9.  (Optional) その後、以下のコマンドで管理画面に入り、上で生成した`ADMIN_TOKEN`でログインします。必要な設定があれば設定します。私の場合は新規登録を不可にし、招待機能をオフにしました。

```bash
flyctl open /admin
```


10. その後Bitwardenクライアントをダウンロードし、左上の矢印をクリックして、デプロイしたアプリケーションのアドレスを入力します。`app-name.fly.dev`、または先ほどアクセスしたWebページを入力すればOKです。

![](image-16.png)

おめでとうございます！ ここまで来られたら、セットアップ完了です！

これからは、なかなか便利なパスワードマネージャーをお使いいただけます。

Appendix 1. OTPはどのように追加しますか？
ID/Password追加時に、以下のようなTOTP欄に

![](image-17.png)
![](image-18.png)


上記のようなSecret Keyを入力すればOKです。

Appendix 2. すでに使っているGoogleパスワードは使えませんか？

A. 可能です！

Googleパスワードマネージャーからパスワードをエクスポートしたあと、

![](image-19.png)

Vaultwardenウェブコンソール -> ログイン -> ツール -> データのインポートを行えば、Chromeで使っていたパスワードをそのまま使えます。

![](image-20.png)
