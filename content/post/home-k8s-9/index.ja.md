---
title: "自宅でRaspberry Piクラスタを使ってデータセンターを構築する - 9. CloudNativePGでKubernetes上のデータベースを運用する"
slug: a4ded8cced5f045c
date: 2023-12-12T22:09:10+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - Raspberry Pi
tags:
    - K8S
    - K3S
    - Raspberry Pi
    - CloudNativePG
    - PostgreSQL
    - Operator
    - HA
---

_注：筆者は韓国在住のため、本文には韓国特有の文脈が含まれることがあります。_

### 序論

データベースは管理が難しいです。

他のPodと違って、データの保管・バックアップ・管理にも気を使わなければなりませんし、フェイルオーバーやパフォーマンスにも注意を払う必要があります。

そのため、他のワークロードはKubernetesクラスタで動かしていても、DBだけはAWS RDSなどのマネージドシステムや別のインスタンスを使うケースもよくあると聞きます。

でも、それって本当に重要でしょうか？自宅でデータセンターを構築するのに、マネージドDBMSの一つくらいあってもいいんじゃないでしょうか？

ということで、作ってみましょう。KubernetesのOperator Patternを利用した [CloudNativePG](https://cloudnative-pg.io/) を使って、Primary 1台、Replica 2台のクラスタをデプロイし、内部ネットワークから接続できるように設定するところまで進めます。

Postgres Operatorではありませんが、[Mysql OperatorでKubernetes環境のMysql DBを運用する](https://nangman14.tistory.com/79#5.%20Mysql%20Operator%20Failover%20%ED%85%8C%EC%8A%A4%ED%8A%B8-1)（韓国語）の記事を読んでいただけると、ついていくのに少し役立つかもしれません。

### 2. インストール

インストールは2段階に分けて進めます。

1. CNPG Operatorのインストール
2. CNPG Clusterのインストール

OperatorはClusterが正常な状態を維持しているかを監視する役割を担います。
実際に使うデータベースクラスタは2.でインストールします。

順を追って始めましょう！今回もArgoCDを使って素早くデプロイします。

**1. CNPG Operatorのインストール**

`apps/enabled/cnpg-system.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cnpg-system
  namespace: argocd
spec:
  destination:
    namespace: cnpg-system
    server: 'https://kubernetes.default.svc'
  source:
    path: modules/cnpg-system
    repoURL: 'git@github.com:<YourOrganizationName>/<YourRepositoryName>.git'
    targetRevision: HEAD
  project: default
```

`modules/cnpg-system/cnpg.yaml`
```yaml
# https://github.com/cloudnative-pg/cloudnative-pg
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cnpg
  namespace: argocd
spec:
  destination:
    namespace: cnpg-system
    server: 'https://kubernetes.default.svc'
  source:
    repoURL: 'https://cloudnative-pg.github.io/charts'
    targetRevision: 0.19.1
    chart: cloudnative-pg
  project: default
```

簡単ですよね？インストール後にデプロイを実行してCNPG Operatorをインストールしましょう。

**2. CNPG Clusterのインストール**

これから作るクラスタの構成は次の通りです。

1. 毎日UTC 0時（KST基準で朝9時）にS3へDaily Backupを実行します。
2. 合計3つのPodで構成されます。各Podは複数のノードに分散され、万が一の不幸（？）を防ぎます。
3. 192.168.0.xのIPで内部ネットワークからDBにアクセスできます。

一つずつ始めましょう！

`apps/enabled/cnpg-cluster.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cnpg-cluster
  namespace: argocd
spec:
  destination:
    namespace: cnpg-cluster
    server: 'https://kubernetes.default.svc'
  source:
    path: modules/cnpg-cluster-16
    repoURL: 'git@github.com:<YourOrganizationName>/<YourRepositoryName>.git'
    targetRevision: HEAD
  project: default
```

`modules/cnpg-cluster/cluster.yaml`
```yaml
# https://cloudnative-pg.io/documentation/1.21/quickstart/
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  namespace: cnpg-cluster
  name: cnpg-cluster
spec:
  instances: 3

  superuserSecret:
    name: superuser-secrets
  enableSuperuserAccess: true
  primaryUpdateStrategy: unsupervised

  # Persistent storage configuration
  storage:
    size: 10Gi
    pvcTemplate:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      storageClassName: longhorn-ssd
      volumeMode: Filesystem

  # Backup properties
  backup:
    retentionPolicy: "90d"
    barmanObjectStore:
      destinationPath: s3://lemon-backup/cnpg-backup
      s3Credentials:
        accessKeyId:
          name: aws-backup-secret
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: aws-backup-secret
          key: ACCESS_SECRET_KEY
      wal:
        compression: gzip
```

管理の利便性のためにSuperuserアクセスを許可し、
DB容量は10GBに設定しました。（後から拡張可能です。）

その他、万が一の事故が起きた時(...)に備えて簡単にバックアップできるよう、バックアップストレージをAWS S3に保存するように設定し、最大90日間保管するようにしました。

`modules/cnpg-cluster/daily-backup.yaml`

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  namespace: cnpg-cluster
  name: daily-backup
spec:
  schedule: "0 0 0 * * *" # Daily
  backupOwnerReference: self
  cluster:
    name: cnpg-cluster
```

シンプルなDaily Backupリソースです。

`modules/cnpg-cluster/lb.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cnpg-lb-rw
  namespace: cnpg-cluster
spec:
  ports:
    - name: postgres
      port: 5432
      protocol: TCP
      targetPort: 5432
  selector:
    cnpg.io/cluster: cnpg-cluster
    role: primary
  type: LoadBalancer
  loadBalancerIP: 192.168.0.206
```

私の場合は192.168.0.206アドレスでアクセスを許可しました。
これにより、データベースの照会や管理時に内部ネットワークから`192.168.0.206`へアクセスして管理できるようになります。

`modules/cnpg-cluster/sealed-aws-secrets.yaml`

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: aws-secrets
  namespace: cnpg-cluster
  annotations: {}
spec:
  encryptedData:
    ACCESS_KEY_ID: adffd...
    ACCESS_SECRET_KEY: Aadfads...
```

AWSでS3FullAccess、または特定バケットへのアクセス権限を与えた後、ACCESS KEYとSecretを発行し、以前設定したSealed Secretを使って登録します。

`modules/cnpg-cluster/sealed-superuser-secrets.yaml`

作成プロセスが少し複雑です！

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: superuser-secrets
  namespace: cnpg-cluster
type: kubernetes.io/basic-auth
stringData:
  username: <使用するID、b64エンコーディングなしの原本そのまま>
  password: <使用するパスワード、b64エンコーディングなしの原本そのまま>
```

上記のようなSecretをまずsecret.yamlという名前で作成した後、

```sh
cat secret.yaml | kubeseal --controller-namespace=sealed-secrets-system --controller-name=sealed-secrets -oyaml > sealed-superuser-secrets.yaml
```

でSealed Secretに変換した後、Sealed Secretを使用します！

その後プロビジョニングを待ち（少し時間がかかります。）、192.168.0.206に先ほど設定したID/PasswordでログインしてDBを使えます！

**正常にインストールされたら、翌日アラームをセットして必ず！！S3の該当フォルダに正常にバックアップされているか確認してください！！！**

### 3. リカバリ

1日経って正常にバックアップされていたら、**必ず！リカバリができるか確認**してください。
データが消えてから確認したのでは遅すぎます...


私のリカバリ設定ファイルを共有します。

```yaml
# https://cloudnative-pg.io/documentation/1.17/quickstart/
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  namespace: cnpg-cluster
  name: cnpg-cluster
spec:
  instances: 3

  superuserSecret:
    name: superuser-secrets
  primaryUpdateStrategy: unsupervised

  bootstrap: # 追加
    recovery:
      source: clusterBackup

  storage:
    size: 10Gi
    pvcTemplate:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      storageClassName: longhorn-ssd
      volumeMode: Filesystem

  externalClusters: # 追加
    - name: clusterBackup
      barmanObjectStore:
        serverName: cnpg-cluster
        destinationPath: s3://lemon-backup/postgres-backup
        s3Credentials:
          accessKeyId:
            name: aws-secrets
            key: ACCESS_KEY_ID
          secretAccessKey:
            name: aws-secrets
            key: ACCESS_SECRET_KEY
        wal:
          compression: gzip
```

新規クラスタに上記のようにbootstrap、externalClustersオプションを追加すると、クラスタが最初に起動する時に既存のS3ファイルを使って自動的にデータをリカバリします。

接続/確認後、リカバリが終わったらbootstrap、externalClustersオプションを削除して、元のバックアップオプションを再度追加すればOKです。

ここで注意すべき点として、
Postgresのメジャーバージョンが異なる場合、リカバリが正常に行われないことがあります。

例えば、私が1.16バージョンのOperatorを使用していて（Postgres 15）、Operatorを1.21（デフォルトでPostgres 16 Operator）まで更新したとしても、既に作成されたクラスタは手動でアップグレードしない限りPG 15を維持します。

この時、もし障害が発生してリカバリを試みる場合、Operatorのバージョンが1.21なのでPG16がプロビジョニングされ、既存のS3に保存されているデータがPG15のデータなのでリカバリできない可能性があります。

このような場合は、Operatorをインストール当時のバージョンにダウングレードして同じバージョンのPGを上げてからリカバリ → アップデートを進めるか、Imageを設定して同じメジャーバージョンを使う方法があります。

**可能であれば、新しいクラスタを一つ作って正常にリカバリできるか必ず確認してから次の内容に進むことを必ず！おすすめします！！**

### 4. クラスタアップデート

マイナーアップデートの場合は自動で進行されますが、メジャーバージョンアップデート時には次のように進めれば、普通は問題ないでしょう。（15 -> 16で実施）

オンラインアップデートの場合は [The Current State of Major PostgreSQL Upgrades with CloudNativePG](https://www.enterprisedb.com/blog/current-state-major-postgresql-upgrades-cloudnativepg-kubernetes) を参考にし、

オフラインアップデート（ダウンタイムが発生しても問題なし）であれば

1. 既存クラスタからpg_dumpallでデータをダンプ
2. 新規クラスタを作成
3. 1でダンプしたデータを2に流し込む
4. 既存クラスタを参照していたアプリケーションを新規クラスタを参照するように変更
5. テスト後、既存クラスタを削除

### 5. 終わりに

私は現在CNPGを使って約7〜8ヶ月間サーバーを正常に運用しています。

掃除中に頻繁にケーブルを抜いてしまうこと(...)を考えると、一般的な状況において十分に堅牢なPostgres DBとして使えますし、誤ってクラスタの電源を全部落とした時も既存のS3から簡単にデータをリカバリできたので、信頼性の高いシステムだなと思いました。

もちろん、お金に余裕があればマネージドRDBを使うのが一番良い選択でしょうが、技術検証も兼ねて、こういうシステムもあるんだということを知っていただければ幸いです！

この時点で、サーバーなどに必要なシステムが全て整い、開発を始められる状態になっているはずです。簡単なことから一つずつ作っていきましょう。

そしてKubernetes内では、このDBはcnpg-lb-rw.cnpg-clusterという名前でアクセスできます。例えば`jdbc:postgresql://cnpg-lb-rw.cnpg-cluster:5432/my_app`のように。

長い記事を読んでいただきお疲れ様でした！次回はnvidia-device-pluginを使ってK3SでGPUを使う方法について見ていきます！
