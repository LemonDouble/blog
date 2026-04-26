---
title: "自分用に残しておくRaspberry Piの初期セットアップ"
slug: 088bdb572b7dc509
date: 2023-01-18T11:13:14Z
image: cover.png
categories:
    - Raspberry Pi
tags:
    - Raspberry Pi
    - SSH
    - Ubuntu
    - netplan
    - 初期設定
---

_注：筆者は韓国在住のため、本文には韓国特有の文脈が含まれることがあります。_

Raspberry Piや自宅サーバーを運用していると、意外と(?)サーバーを初期化しなければならない場面が多くあります。

設定をミスして何かがうまく動かないとか、何をインストールしたのかも分からないくらい色々入っている状態になっているとか。

そういうわけで、Raspberry Piを初期化したときに毎回やる一連の作業があるのですが、

困ったことに、いつも忘れた頃に初期化することになるので、セットアップのたびにGoogleで延々と調べ直すことになります。なので、ここにまとめておこうと思います。


`以下の内容はRaspberry Pi + Ubuntu Serverを前提としています。初期ログインパスワードは ubuntu です。`


### 1. SSH接続を有効にする

* HDMIケーブルを持ってきて差して、キーボードまで繋ぐのは本当に面倒です。
* ブートディスクを焼くときに `ssh`(拡張子なし)という空ファイルを1つ作っておくと、起動時にSSHアクセスが有効になります。(デフォルトでは無効)
* PiをLANケーブルでルーターに接続し、ルーターの管理画面などからIPを確認してSSH接続します。hostnameは `ubuntu` になります。

### 2. SSHログイン設定

* SSHログインができるように、`~/.ssh/authorized_keys` ファイルに自分の公開鍵を追加しましょう。
* SSHキーがない場合は「SSH 鍵 ログイン」などでGoogle検索して手順を辿ってみてください！

### 3. パスワードログインを無効化する

* 内部ネットワークでしか公開していないPiならあまり気にしなくてもいいですが…SSH Keyがあるなら、パスワードログインはなるべく無効化しておく方がセキュリティ的に安心です。
* `/etc/ssh/sshd_config` を開いて `PasswordAuthentication no` に設定します。
* `/etc/ssh/sshd_config.d/50-cloud-init.conf` を開いて、こちらも `PasswordAuthentication no` に設定します。
* その後 `sudo systemctl reload sshd` でSSHサーバーを再起動します。

### 4. Wi-Fi接続(任意)

* LANケーブルで使い続けるならやらなくても大丈夫です。私はLANケーブルがあると部屋が散らかるので、Wi-Fiを設定しています。
* `/etc/netplan/50-cloud-init.yaml` ファイルを開きます。(なければ作成します。)
* 下記のファイルをコピペします。下の設定ファイルはSSID(Wi-Fi名) = lemon、パスワードは lemon1234 の例です。

```yaml
# This file is generated from information provided by the datasource.  Changes
# to it will not persist across an instance reboot.  To disable cloud-init's
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
    ethernets:
        eth0:
            dhcp4: true
            optional: true
    wifis:
      wlan0:
        dhcp4: true
        optional: true
        access-points:
          "lemon":
            password: "lemon1234"
    version: 2
```

### 5. Hostnameの設定

* SSH画面で今何を触っているか分からないと、なんだか不安になります…
* `sudo hostnamectl set-hostname <ホスト名>` を入力して、hostnameも分かりやすい名前に変更しておきます。


### 6. 再起動

* `sudo reboot`
* 無線LANで再接続するときに内部IPが変わる場合があるので、繋がらなければルーターの画面を見て内部IPを修正します。
