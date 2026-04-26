---
title: "自宅でラズベリーパイクラスタを使ってデータセンターを構築する - 序論"
slug: d42387934c57aa25
date: 2023-12-03T13:24:05+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - ラズベリーパイ
tags:
    - K8S
    - K3S
    - ラズベリーパイ
    - クラスタ構築
    - ホームラボ
    - アーキテクチャ
---

_注：筆者は韓国在住のため、本文には韓国特有の文脈が含まれることがあります。_

### 初めてラズベリーパイクラスタを構築してから、おおよそ300日が経ちました。

きっかけはそれほど大きなものではありませんでした。

私は毎朝[GeekNews](https://news.hada.io/)[^1]というニュースキュレーションサービスを見ながら出勤しているのですが、その中で[Chick-Fil-A の Edge Computing 技術アーキテクチャ：Enterprise Restaurant Compute](https://news.hada.io/topic?id=8306)という記事が目に留まりました。

そこから「チックフィレイもKubernetesを動かしているのに、サーバーエンジニアとして自分も管理するクラスタの一つくらいあってもいいんじゃないか?」というのが、すべての始まりでした。

### 構成環境

私の屋上部屋(...)のクラスタはおおよそこのような感じです。
![Alt text](image.png)
![Alt text](image-1.png)

構成スペックを整理すると、

- **Control Node**
  - Raspberry PI 4b+ 8GB Model * 1
  - Samsung MUF-AB FIT PLUS 64GB USB
- **ARM Worker + Storage Node**
  - Raspberry PI 4b+ 8GB Model + 500GB SSD(PNY CS900 500GB) * 3
  - Sandisk USB Ultra Fit USB 3.1 32GB * 3
- **GPU X86 Worker Node (For ML)**
  - Ryzen 5600x + 64GB DDR4 3200 RAM + RTX3090 + NVME SSD(WD Black) 1TB + WD RED Plus 4TB HDD

このように構成し、ラズベリーパイ4台、X86 GPUサーバー1台で、X86・ARM混合クラスタを運用しています。

ラズベリーパイのディスクI/Oは安定性のため、SDカードの代わりにUSBディスクを使用しています。

### あれ?Control Nodeが1台だけならHAできないんじゃないですか?

- 最初のクラスタ構築時の最大の悩みでした。
  - 3台以上をHAで構成すれば安全ですが、Piの演算能力がそれほど高くないことを考えると、できるだけ無駄なコンピューティングパワーを減らしたいと思いました。

- その結論として、**Master Nodeは1台**にしたうえで、以下のような安全装置を用意しています。
  - etcdの代わりに**External Database**を状態保存先として使用しています。
    - 現在は**Supabase**が提供するPostgresqlを外部ストレージとして使用しています。
  - Master NodeはTaintをかけて、Job schedulingができないように設定しています。
    - これにより、作業割り当て中にマスターノードが突然停止するなどの事態を減らしました。
  - Master Nodeには少しだけ信頼性の高い機材を使用しています(USB、クーラーなど)。

それにもかかわらず、夏にMaster Nodeが過熱で(...)USBが故障したことがありましたが、すべてのStateが外部DBに存在するため、10分以内に復旧することができました。

### 現在ソフトウェア的に構築した内容

1. K3SとExternal DBを利用して、もしMaster Nodeがダウンしても復旧できるようにシステムを構築します。
2. ArgoCDとGithubを利用して、GitOpsシステムを構築します。
3. [Mend Renovate](https://www.mend.io/renovate/)を利用して、インストールしたHelm ChartおよびPrivate Registryの新しいバージョンが出たら自動的にPRを作成し、クラスタを最新状態に保ちます。
4. [Longhorn](https://github.com/longhorn/longhorn)を利用して分散ストレージを実装し、もしSSDが1台故障しても復旧できるシステムを構築します。
5. Docker Private Registryを構築し、[Docker-registry-browser](https://github.com/klausmeyer/docker-registry-browser)を利用してアップロードされたイメージを確認できるGUIを構築します。
6. [Sealed-secrets](https://github.com/bitnami-labs/sealed-secrets)を利用してSecretを外部サービスなしでGitに保存して管理します。また、[Kubeseal-webgui](https://github.com/Jaydee94/kubeseal-webgui)を利用してGUIで便利にSecretを追加できる画面を作ります。
7. [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)を利用してモニタリングシステムを構築・管理します。
8. [CloudNativePG](https://cloudnative-pg.io/)を利用してHA Postgresデータベースシステムを構築し、万が一の事故を防ぐためにAWSなどへ自動バックアップシステムを構築します。
9. [Portainer](https://www.portainer.io/)を利用してWeb UIベースでクラスタ管理を行うシステムを作ります。
10. [MetalLB](https://metallb.universe.tf/)の設定により、内部ネットワークからのみアクセス可能なエンドポイント、サービスを簡単に構築します。
11. [nvidia-device-plugin](https://github.com/NVIDIA/k8s-device-plugin)を利用してKubernetes上でコンテナがCUDAなどのデバイスを使用できるようにします。
12. [traefik-forward-auth](https://github.com/thomseddon/traefik-forward-auth)ライブラリのアイデアを借用して、独自の認証システムを構築します。パスワード以外にSSOログインを通じて内部管理システムにアクセスできるようにします。

### これから書く記事

これからは、私が構築した内容を振り返りながら、ハードウェアからソフトウェア、システム構築までの内容を時間があるときに一つずつ追加していく予定です。
上記は順不同で並べた内容であり、書き進めるうちに内容が変わる可能性があります。

なお、このセットアップガイドは2023年12月時点で、私が直接実際にたどって正常動作を確認した方法です。

### 序論を終えて

クラスタを構築しながら関連資料を一生懸命探しましたが、

意外にもこの種の資料はそれほど多くないことに気づきました。
[VladoPortos](https://github.com/VladoPortos)さんの[Kubernetes with OpenFaaS on Raspberry Pi 4](https://rpi4cluster.com/)の資料は非常に役立ちましたが、国内資料がそれほど多くないのが残念でした。


微力ではありますが、このガイドが似た道を切り拓こうとする方の助けになれば幸いです。

[^1]: GeekNewsは韓国の技術系ニュースキュレーションサイトで、Hacker Newsに似ていますが韓国の開発者コミュニティ向けです。
