---
title: "Raspberry Pi + Ubuntu 22.04にK3Sをインストールして使う"
slug: fe0cac9de367fc18
date: 2023-01-18T11:56:02Z
image: cover.png
categories:
    - K8S
    - K3S
    - Raspberry Pi
tags:
    - Raspberry Pi
    - K3S
    - Ubuntu
    - インストールガイド
---

_注：筆者は韓国在住のため、本文には韓国特有の文脈が含まれることがあります。_

最近とてもホットでありながら難しい技術がKubernetesだと思います。

私は学生時代からそれなりに長く(?)ホームサーバーを運用してきましたが、運用しながら結構大変なことに何度も直面しました。

引っ越してネットワークを組み直さなければならなかったり、ディスクが1台壊れたりすることもたまにあって、そのたびにGitや他のコンピュータを探ってサイトを復旧するのは本当に...面倒な作業です。

「自分のサーバーがどう動いているか書き留めておいて、いつでも好きな時にコンピュータを追加したりディスクを追加したりできたらどんなに良いだろう？」と思っていた中で、K8Sはとても魅力的な選択肢でした。

それでK8Sを勉強してみたのですが、たいていはVagrantなどを使って高性能なコンピュータ1台に仮想ノードを1つ置き、仮想VMを立ち上げて実習する形になります。

`ところが問題はその後です。`

デスクトップのホームサーバーを何台も買ってK8Sを動かすわけにはいかないので、`安くて && 低消費電力のPC`を探すことになり、たいていはRaspberry Piに行き着きます。

ところが、講師の先生がきれいに用意してくれたインストールスクリプトを使うことに慣れていると、ARMに直接K8Sをインストールしようとすると頭が爆発します。(実際私は爆発しました。他の人はわかりませんが...)

[KubeAdm](https://github.com/kubernetes/kubeadm)、[MicroK8S](https://github.com/canonical/microk8s)を何度か試行錯誤した末にインストールに失敗しましたが、K3Sはとても簡単にインストールできたので、その経験を共有しようと思います。

![](image1.png)


`以下の環境はRaspberry pi 4B + Ubuntu Server 22.04で行いました。`

### 1. Master Nodeのインストール

* K3Sの[公式ドキュメント](https://docs.k3s.io/quick-start)を見ると、`curl -sfL https://get.k3s.io | sh -`を実行するだけでいい感じにインストールしてくれるそうです！
* bashに`curl -sfL https://get.k3s.io | sh -`を入力してインストールしてみます。
* おお！インストールがうまくいきます。
* kubectlも一緒にインストールしてくれるそうです。`kubectl get nodes`を入力してみます。
* おお！ノードもちゃんと表示されます。

### でも実は罠です。

* `kubectl get nodes`を
* あれ？急に応答が遅いです。
* `The connection to the server 127.0.0.1:6443 was refused - did you specify the right host or port?`という問題に遭遇します。
* ああ...これも罠なのか？不安になります。
* 本当に幸いなことにこれはKnown Issueで、解決方法もあります。(関連[Github Issue](https://github.com/k3s-io/k3s/issues/4234))
* `sudo apt install linux-modules-extra-raspi && reboot`を入力して追加で必要な依存関係をインストールし、再起動した後に`kubectl get nodes`を実行するとうまくいきます。(公式ドキュメント[Link](https://docs.k3s.io/advanced#raspberry-pi))

### 2. ノードを追加する

* Master Nodeを設定したので、Worker Nodeを追加してみましょう。
* `kubectl get nodes`を入力して、Master Nodeの名前(Name)を確認します。
* `sudo kubectl get node <master nodeの名前> -ojsonpath="{.status.addresses[0].address}"`を入力して、表示されるIP値を確認します。(1)
* `sudo cat /var/lib/rancher/k3s/server/node-token`を入力して、表示されるToken値を確認します。(2)

* 新しいWorker Nodeに接続して`sudo apt install linux-modules-extra-raspi && reboot`を実行し、関連する依存関係をインストールします。
* その後`curl -sfL https://get.k3s.io | K3S_URL=https://<(1)で確認したIP>:6443 K3S_TOKEN=<(2)で確認したトークン> sh -`を実行します。
* その後Master Nodeで`kubectl get nodes`を入力して、正常にクラスターにJoinされたか確認します。

* 追加：Worker NodeのステータスがNotReadyから進まない場合があるのですが、このケースに対応してみましょう。
    *  `kubectl get nodes`を入力してWorker Nodeの名前を覚えます。
    *  `sudo kubectl describe node <worker nodeの名前> -n=kube-system`を入力して現在のノードの状態を確認します。
    *  私の場合は`invalid capacity 0 on image filesystem`というエラーが続けて発生してWorker Nodeがjoinできない状況でしたが、再起動したら解決しました(..)

以上！
