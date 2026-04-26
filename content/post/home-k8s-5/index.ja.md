---
title: "自宅でRaspberry Piクラスターを使ってデータセンターを作ってみる - 5. ArgoCD GitOps設定とLonghornインストール"
slug: 107eed6a453d6238
date: 2023-12-09T09:16:13+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - Raspberry Pi
tags:
    - K8S
    - K3S
    - Raspberry Pi
    - ArgoCD
    - GitOps
    - Longhorn
    - ストレージ
---

_注：筆者は韓国在住のため、本文には韓国特有の文脈が含まれることがあります。_

### 1. GitOpsを始める

ArgoCDをインストールしましたので、リポジトリ構造を整理してみましょう。

App On Appsパターンを利用して、シンプルなデプロイ方法を作っていきます。

理解できなくても、まずは付いてきてください。

前回作成した空のリポジトリを、次のような構造にしてください。

```sh
/apps
    ㄴ/disabled
    ㄴ/enabled
/init
/modules
```

その後、initフォルダに以前作業した ip_address_pool.yaml、(ArgoCDの) ingress.yaml、traefik-update.yaml、values.yaml をすべて追加します。

最初の設定時のファイルをGitに保管しておくことで、後でクラスターを復旧する状況になったときに、これらの設定ファイルを `kubectl -f ファイル名` で素早く適用できるようにするためです。

下記のようにファイルを追加し、これまでに行った作業を書いた README.md を1つ追加しておくと、クラスターに問題が発生したときの復旧に大いに役立ちます。README.md はご自身で書いてください！

```sh
/init
    ㄴ ip_address_pool.yaml
    ㄴ ingress.yaml
    ㄴ traefik-update.yaml
    ㄴ values.yaml
```

その後、initフォルダ内に以下のyamlファイルを参考に root-app.yaml を追加します。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <お好きな名前 (筆者の場合は lemon-k8s-apps)>
  namespace: argocd
spec:
  destination:
    namespace: argocd
    server: 'https://kubernetes.default.svc'
  source:
    repoURL: 'git@github.com:<YourOrganizationName>/<YourRepositoryName>.git'
    targetRevision: HEAD
    path: apps/enabled
  project: default
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

一つずつ見ていくと

1. ArgoCDが管理するアプリケーションを1つ追加
2. アプリケーションは GitHub の自分のリポジトリを参照
3. Path は上で設定した apps/enabled の中にあるすべての yaml ファイルを実行

となります。

その後 `kubectl apply -f root-app.yaml` を実行して root app をインストールします。

そして ArgoCD に戻り、自分の最初のアプリケーションがちゃんと表示されているか確認します。Sync が失敗し、リポジトリにそのパスが存在しないと表示されていれば正常に接続されています。

### 2. Longhornのインストール

* メインUSB(OSが入っているUSB)以外に、最低1つ以上のストレージ用USB、SSD、HDDが接続されている前提で進めます。

**2023年12月時点で、ArgoCD で初回インストールすると Longhorn の初期 Batch Job が失敗するバグがあるため、手動で1度インストールしてから ArgoCD に移行する作業を行います。**

Longhorn は分散ストレージマネージャで、次のような機能を提供します。(読むのが面倒なら、ざっくり S3 に近い機能を提供すると思ってください！)

1. **分散ストレージ保存**

ファイルを設定に応じて N 個のストレージに分散保存します。

何らかの障害でストレージが1つ壊れても、そのストレージにあったデータを他のストレージへ自動的に移行し、最低 N 個のレプリカが維持されるようにします。

これにより、SSDが1つ壊れても他のストレージに同じデータがあるので、プログラムやシステムを正常に運用し続けることができます。

2. **Storage Tiering**

あるシステムは I/O が非常に重いワークロードで、高いストレージ性能が必要な場合があり、

別のシステムは滅多に読まないファイルが大量にあって、性能より容量が重要な場合があります。

Longhorn は StorageClass をサポートしており、これによってストレージを段階別に分けて使うことができます。

例えば I/O が重要な場合は NVMe SSD のみで構成したストレージクラスを、単純なファイル保存が中心の場合は HDD のみのストレージクラスを割り当てる、といった使い方をします。

というわけで！次のコマンドで Longhorn をインストールします。

```sh
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace --set service.ui.loadBalancerIP="192.168.0.201" --set service.ui.type="LoadBalancer"
```

正常にインストールされたら、ブラウザで `http://192.168.0.201` にアクセスして、Longhorn UI が表示されることを確認します。

### 3-1. ArgoCDを使ってインストールする - 設定ファイルの追加

一度インストールが完了しましたので、すでにインストール済みの Longhorn を ArgoCD に移していきましょう。

フォルダ構造とデプロイ戦略は次のとおりです。下記のような構造で、root-app をデプロイすると下位アプリが自動的に立ち上がるようにします。

![フォルダ構造、Root App が enabled フォルダをデプロイし、enabled フォルダ下の yaml ファイルが modules/ファイル名 をデプロイする](image.png)

これも同様に、まずは真似してみましょう。下記ファイルを追加します。

* `apps/enabled/longhorn-system.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: longhorn-system
  namespace: argocd
spec:
  destination:
    namespace: longhorn-system
    server: 'https://kubernetes.default.svc'
  source:
    path: modules/longhorn-system
    repoURL: 'git@github.com:<YourOrganizationName>/<YourRepositoryName>.git'
    targetRevision: HEAD
  project: default
```
modules/longhorn-system を自動デプロイするアプリケーションを追加します。

* `modules/longhorn-system/longhorn.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: longhorn
  namespace: argocd
spec:
  destination:
    namespace: longhorn-system
    server: 'https://kubernetes.default.svc'
  source:
    repoURL: 'https://charts.longhorn.io'
    targetRevision: 1.5.3
    chart: longhorn
    helm:
      parameters:
        - name: service.ui.loadBalancerIP
          value: 192.168.0.201
        - name: service.ui.type
          value: LoadBalancer
  project: default
```

`helm.parameters` を使うことで、上記のように values.yaml に入れる値を追加できます。


* `modules/longhorn-system/storage-class.yaml`

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: longhorn-ssd
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: "Delete"
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "2"
  fsType: "ext4"
  diskSelector: "ssd"
---
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: longhorn-hdd
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: "Delete"
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "1"
  fsType: "ext4"
  diskSelector: "hdd"
```
SSD、HDD 用ストレージを追加します。USB などを使っている場合は、名前を longhorn-usb などに変えても構いません。

`numberOfReplicas`：レプリカの数です。筆者の場合、SSD はアプリケーションに必要なデータを保存するので、安定性のために最低2個、HDD は単純なファイル保存用なので1個に設定しました。
`fsType`：ファイルシステムのタイプです。ext4 を使用する予定です。ガイド通りに進めるなら、同じ値で大丈夫です。
`diskSelector`：後でディスク登録時に使います。覚えておいてください。

作業が完了したら、Git commit のあと Push します！

### 3-2. ArgoCDを使ってインストールする - 実際にデプロイする

それでは、最初に設定した ArgoCD に入って、実際のデプロイを行っていきます。

![ArgoCD Root App の画像](image-1.png)

ここで知っておくべきは `Sync`、`Refresh` ボタンです。

- `Sync`：Git から新しいクラスター状態を取得した後、実際のクラスター状態を Git と同期します。
- `Refresh`：Git から新しいクラスター状態を取得します。クラスターの同期は行いません。

つまり、Sync を押すと ArgoCD がデプロイを開始します。

その後 Longhorn-system -> Longhorn の順にたどっていき、Sync ボタンを押してデプロイを進めます。

### おわりに

お疲れさまでした！ようやく今まであちこちに散らばっていた YAML ファイルを整理し、GitOps に向けて第一歩を踏み出しました。

次回は実際の SSD、HDD を Longhorn を使って登録してみます。
