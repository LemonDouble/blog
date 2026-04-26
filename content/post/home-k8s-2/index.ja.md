---
title: "自宅でラズベリーパイクラスタによるデータセンター構築 - 2. クラスタ初期設定編"
slug: f4efced2e8d5b1d8
date: 2023-12-03T22:20:29+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - ラズベリーパイ
tags:
    - K8S
    - K3S
    - ラズベリーパイ
    - Ubuntu
    - DHCP
    - Supabase
    - PostgreSQL
    - クラスタ構築
---

_注：筆者は韓国在住のため、本文には韓国特有の文脈が含まれることがあります。_

### ハードウェア

![ハードウェア構成図](image.png)

上の図を参考にハードウェアを構成します。

普段からPCを使っているご家庭なら、ルーターと普段使いのデスクトップはすでにあると思いますので、図のうち緑色の部分だけを追加すれば大丈夫です。

ネットでスイッチハブ（アンマネージドスイッチ）を購入したら、ラズベリーパイのLANケーブルをすべてスイッチハブに繋ぎ、ルーターからインターネットケーブルを1本引いてきてスイッチに繋ぐだけです。

### ラズベリーパイの設定

以前公開した記事 [自分用に書くラズベリーパイ初期セットアップ](https://lemondouble.github.io/ja/p/%EB%82%B4%EA%B0%80-%EB%B3%B4%EB%A0%A4%EA%B3%A0-%EC%98%AC%EB%A6%AC%EB%8A%94-%EB%9D%BC%EC%A6%88%EB%B2%A0%EB%A6%AC-%ED%8C%8C%EC%9D%B4-%EC%B4%88%EA%B8%B0-%EC%84%B8%ED%8C%85/) と、[SDカードを3枚も飛ばしてキレて書くラズベリーパイのUSB Boot設定方法](https://lemondouble.github.io/ja/p/sd%EC%B9%B4%EB%93%9C-%EC%84%B8%EA%B0%9C-%EB%82%A0%EB%A0%A4%EB%A8%B9%EA%B3%A0-%ED%99%94%EB%82%98%EC%84%9C-%EC%93%B0%EB%8A%94-%EB%9D%BC%EC%A6%88%EB%B2%A0%EB%A6%AC%ED%8C%8C%EC%9D%B4-usb-boot-%EC%84%B8%ED%8C%85-%EB%B0%A9%EB%B2%95/)、もしくは他のブログ記事を参考にして、ラズベリーパイにUbuntu Server 22.04をインストールします。

SDカードでもクラスタ構築は可能ですが、USB Bootを設定したうえでUSBから起動することを**強くおすすめ**します。

ここまで来たら、ラズベリーパイにUbuntuがインストールされていてSSH接続ができる前提で進めます。

その後、`sudo apt update -y && sudo apt upgrade -y && sudo apt install linux-modules-extra-raspi` で必要な依存関係をインストールします。（[ラズベリーパイ + Ubuntu 22.04にK3Sをインストールして使う](https://lemondouble.github.io/ja/p/%EB%9D%BC%EC%A6%88%EB%B2%A0%EB%A6%AC-%ED%8C%8C%EC%9D%B4--ubuntu-22.04%EC%97%90-k3s-%EC%84%A4%EC%B9%98%ED%95%B4%EC%84%9C-%EC%93%B0%EA%B8%B0/) を参照）

### DHCPサーバーの設定

その後、最初に一度すべてのパイの電源を入れると、各パイは内部IPを割り当てられます。（設定を変えていなければ192.168.0.Xの形）

何もしないと、この内部アドレスはパイの電源が入るたびに**空いているIPの中の1つ**を取って設定されるので、IPが流動的に変わってしまい、毎回ルーター管理ページなどでノードのIPを確認する手間が発生します。

~~（正確には、変わるかもしれないし変わらないかもしれません。たいていそのまま維持しようとはしますが、念のため事前に設定しておきます。）~~

![ipTIMEのDHCPサーバー設定](image-1.png)

上のように、DHCPサーバー設定（ipTIME[^1]を例として）と似たメニューを探し、最初に接続したパイの**MACアドレス**（1A-2B-3C...）を入力したうえで、特定のIPを割り当てます。

私は便宜上、192.168.0.20をControl Node、192.168.0.21〜30をWorker Nodeとしましたが、IPはお好きに設定してください。

その後、WindowsユーザーであればPutty、LinuxやMacのユーザーであればssh CLIを使って接続準備をします。

### Supabaseの登録とPostgres接続権限の取得

Control Nodeを3台割り当てるのはリソースがもったいなく、1台だけだとControl Nodeが落ちたときの復旧が難しいので、その妥協案として、

万が一Control Nodeが落ちても、ステートを外部SaaSに保存しておくことで、Control Nodeを交換すれば外部Storageからステートを取得して使えるようにします。

これに無料DBを提供する**Supabase**を使います。

![500MBまで無料！](image-2.png)

[Supabase](https://supabase.com/) に登録し、Postgres接続権限を設定して取得します。

できれば、SupabaseはPgBouncerもサポートしているのでPgBouncerも設定しておくと良いです。

（同時接続数の制限があるため、後でPortainerをインストールすると動作はしますが、PgBouncerなしだとエラーメッセージが頻発します。）

### Master Nodeの設定

*上記の手順に従ってUbuntuがインストールされている前提です。*

以下を準備します。

1. Master Nodeとして使うラズベリーパイのIP（上のDHCPサーバー設定で割り当てたIP）
   - 例：`192.168.0.20` のような形。**後でWorker Nodeを参加させるときに必要なので、必ず控えておいてください。**
2. Kubernetesクラスタが互いにJoinするときに使うランダムトークンを用意します。私は記号なしの英大文字小文字＋数字の組み合わせ60文字のランダム文字列を使いました。**後でWorker Nodeを参加させるときに必要なので、必ず控えておいてください。**
   - 例：`jUofPu4hMegXLDKgCn6FbsfWcL8mFQu3DYEiyDjQgh8447cMAqwfgYwTeNU4`
3. 上で用意したPostgresエンドポイントを準備します。
   - 例：`postgres://postgres:postgresPassword@db.mydb.supabase.co:5432/postgres`

```sh
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644 --disable servicelb --token <2,myToken> --node-taint CriticalAddonsOnly=true:NoExecute --bind-address <1,myControlNodeIP>  --disable local-storage --datastore-endpoint <3, postGresEndpoint>
```

1, 2, 3に合わせて以下のように設定します。下記は例です。
```sh
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644 --disable servicelb --token mytoken --node-taint CriticalAddonsOnly=true:NoExecute --bind-address 192.168.0.20  --disable local-storage --datastore-endpoint postgres://postgres:postgresPassword@db.mydb.supabase.co:5432/postgres
```

オプションを1つずつ見ると次のとおりです。

1. `curl -sfL https://get.k3s.io | sh -s -`：K3Sをセットアップします。
2. `--write-kubeconfig-mode 644`：KubeConfigファイルのアクセス権を644に設定します。後の設定で必要になります。
3. `--disable servicelb`：K3Sのデフォルト [Service Load Balancer](https://docs.k3s.io/networking#service-load-balancer)（klipper-lb）を無効化します。後でMetalLBというロードバランサを別途インストールして使用します。
4. `--token <mytoken>`：K3Sクラスタが互いにJoinするときに使うトークンを設定します。
5. `--node-taint CriticalAddonsOnly=true:NoExecute`：Control NodeにTaintを追加します。つまり、**安定性のために本当に重要なコンテナ以外はControl Nodeで実行しないようにします。** ノードが1台だけだったり、Control Nodeでもコンテナを実行したい場合は、このオプションを外してください。
6. `--bind-address <myControlNodeIP>`：マスターノードを特定のIPに紐付けるフラグです。
7. `--disable local-storage`：K3SのデフォルトのLocal Storageを無効化します。後でLonghorn Storageを別途インストールして使用します。
8. `--datastore-endpoint <postGresEndpoint>`：内蔵Etcdの代わりに外部Databaseをステートストアとして使用します。

このコマンドをMaster Nodeで実行するとKubernetesのインストールが始まります。

**続いて、便宜上Hostsファイルを編集します。**

`sudo vi /etc/hosts` を実行（テキストエディタは何でも構いません）し、次のような行を追加します。

```sh
192.168.0.20 control01 control01.local

192.168.0.21 worker01 worker01.local
192.168.0.22 worker02 worker02.local
192.168.0.23 worker03 worker03.local
```

私の場合、192.168.0.20をcontrol01というニックネームで、21をworker01、22をworker02というニックネームで使いました。

このように設定しておくと、以降control NodeからSSH接続するときに `ssh worker01` のように簡単に接続できるようになります。

**最後に、Master NodeからWorker Nodeに簡単にアクセスできるよう、SSHを設定します。**

`ssh-keygen` コマンドを入力し、何も入力せずEnterだけ押してssh keyを作成します。
その後 `~/.ssh/id_rsa.pub` ファイルを丁寧にコピーしておきます。

### Worker Nodeの設定

*上記の手順に従ってUbuntuがインストールされている前提です。*


**まずSSH接続から設定します。**

`~/.ssh/authorized_keys` に、改行を1つ入れてから先ほどコピーしたid_rsa.pubの内容を貼り付けます。

その後、control Nodeから `ssh worker01` などを入力して接続できるか確認します。

うまく接続できない場合は、ssh authorized_keysの登録などで検索して解決してください。

**続いてK3Sをインストールします。**

以下を準備します。

1. 上で設定したMaster NodeのIP
2. 上で設定したMaster Nodeのトークン

その後、以下のコマンドを入力してWorker NodeをJoinさせます。

```sh
curl -sfL https://get.k3s.io | K3S_URL=https://<1,myControlNodeIP>:6443 K3S_TOKEN=<2,myToken> sh -
```

1, 2に合わせて上のように設定します。下記は例です。

```sh
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.0.20:6443 K3S_TOKEN=mytoken sh -
```

その後、Node Joinを確認します。

### ノードJoinの確認とラベル作成

その後、Control Nodeで `kubectl get nodes` を入力して、ノードが正常にJoinしていることを確認します。

続いて、ROLES部分を修正するため、Control Nodeで各ノードに対し以下のコマンドを実行します。

ノードが少なければ少ない数だけ、多ければ多い分だけ実行してください。

```sh
kubectl label nodes worker01 kubernetes.io/role=worker
kubectl label nodes worker02 kubernetes.io/role=worker
kubectl label nodes worker03 kubernetes.io/role=worker
```

その後、コンテナ作成時に特定のPod（Worker Node）にだけ立ち上がるように設定するためのラベルも追加します。（上下の両方を実行する必要があります！）
```sh
kubectl label nodes worker01 node-type=worker
kubectl label nodes worker02 node-type=worker
kubectl label nodes worker03 node-type=worker
```

最終確認を行います。

以下は私の場合の出力です。皆さんの環境ではROLESがまだ違っているかもしれません。（`kubectl get nodes`）

```sh
NAME           STATUS   ROLES                  AGE    VERSION
control01      Ready    control-plane,master   307d   v1.27.4+k3s1
worker02       Ready    worker                 307d   v1.27.4+k3s1
worker01       Ready    worker                 307d   v1.27.4+k3s1
worker03       Ready    worker                 307d   v1.27.4+k3s1
```

また、追加したラベルも確認します。（`kubectl get nodes --show-labels`）

ラベルがたくさんあるのは普通なので、追加した `node-type=worker` がきちんと付いているかだけ確認すればOKです。

### おわりに

おめでとうございます！ハードウェアおよびKubernetesのインストールを無事に終えました。

長い設定なので、進めていて詰まる部分があれば気軽にコメントで教えてください！

[^1]: ipTIMEは韓国でよく使われている家庭用ルーターブランドで、韓国の家庭ユーザーにとって馴染み深い管理画面です。同じDHCP予約の考え方は他社ルーターでも適用できます。
