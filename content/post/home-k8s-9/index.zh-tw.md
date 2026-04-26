---
title: "在家用樹莓派叢集架設資料中心 - 9. 使用 CloudNativePG 在 Kubernetes 上運維資料庫"
slug: a4ded8cced5f045c
date: 2023-12-12T22:09:10+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - 樹莓派
tags:
    - K8S
    - K3S
    - 樹莓派
    - CloudNativePG
    - PostgreSQL
    - Operator
    - HA
---

_註：筆者居住於韓國，部分內容包含韓國特有的背景。_

### 序言

資料庫管理起來相當棘手。

與其他 Pod 不同，必須關心資料保管、備份、管理，還要留意 Failover 和效能。

所以據我所知，即使其他工作負載都跑在 Kubernetes 叢集上，單獨把 DB 放在 AWS RDS 之類的託管系統或獨立執行個體上的情況也很常見。

但這真的重要嗎？要在家裡架設資料中心，難道不該有一台託管 DBMS 嗎？

那就來做一個吧。我們將利用 Kubernetes 的 Operator Pattern，透過 [CloudNativePG](https://cloudnative-pg.io/) 部署一個 1 Primary、2 Replica 的叢集，並設定成可以從內網存取。

雖然不是 Postgres Operator，但若讀過 [用 Mysql Operator 在 Kubernetes 環境運維 Mysql DB](https://nangman14.tistory.com/79#5.%20Mysql%20Operator%20Failover%20%ED%85%8C%EC%8A%A4%ED%8A%B8-1)（韓文）這篇文章，對跟上節奏可能會有些幫助。

### 2. 安裝

安裝分成兩個步驟進行。

1. 安裝 CNPG Operator
2. 安裝 CNPG Cluster

Operator 的作用是監視 Cluster 是否維持正常狀態。
實際使用的資料庫叢集則透過第 2 步安裝。

讓我們一步步開始吧！這次也使用 ArgoCD 快速部署。

**1. 安裝 CNPG Operator**

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

簡單吧？安裝後執行部署，把 CNPG Operator 安裝起來。

**2. 安裝 CNPG Cluster**

我們要建立的叢集樣貌如下：

1. 每天 UTC 0 點（KST 上午 9 點）執行 S3 Daily Backup。
2. 總共由 3 個 Pod 組成。每個 Pod 分散在多個節點上，以預防可能的不幸（？）。
3. 透過 192.168.0.x 的 IP 可從內網存取 DB。

讓我們一個一個開始吧！

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

為了管理方便，開放了 Superuser 存取，
DB 容量設為 10GB。（之後可以擴充。）

此外，為了在萬一發生意外事故時(...)能輕鬆備份，將備份儲存設定為儲存到 AWS S3，並保留最多 90 天。

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

簡單的 Daily Backup 資源。

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

我這邊允許透過 192.168.0.206 位址存取。
之後在查詢或管理資料庫時，可以從內網透過 `192.168.0.206` 進行存取與管理。

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

在 AWS 上授予 S3FullAccess，或對特定 Bucket 的存取權限後，發行 ACCESS KEY 與 Secret，再以先前設定的 Sealed Secret 註冊。

`modules/cnpg-cluster/sealed-superuser-secrets.yaml`

建立流程稍微有點複雜！

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: superuser-secrets
  namespace: cnpg-cluster
type: kubernetes.io/basic-auth
stringData:
  username: <我要使用的 ID，不要 b64 編碼，原樣填入>
  password: <我要使用的密碼，不要 b64 編碼，原樣填入>
```

先按上面的內容以 secret.yaml 為名建立 Secret，再用

```sh
cat secret.yaml | kubeseal --controller-namespace=sealed-secrets-system --controller-name=sealed-secrets -oyaml > sealed-superuser-secrets.yaml
```

將其轉換成 Sealed Secret，然後使用 Sealed Secret！

之後等待 Provisioning（需要一些時間），即可透過 192.168.0.206 用剛才設定的 ID/Password 登入使用 DB！

**若安裝正常，請務必！設定隔天的提醒，確認 S3 對應資料夾是否正常備份了！！！**

### 3. 還原

過了一天確認備份正常後，**請務必！確認是否能還原**。
等資料消失後再確認就太遲了……


分享一下我的還原設定檔。

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

  bootstrap: # 新增
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

  externalClusters: # 新增
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

按上面的方式將 bootstrap、externalClusters 選項加入新叢集後，叢集首次啟動時就會用既有的 S3 檔案自動還原資料。

連線/確認後還原完成，再把 bootstrap、externalClusters 選項移除，並把原本的備份選項加回來即可。

此處需要注意的是，
若 Postgres 主要版本不同，還原可能無法正常進行。

舉例來說，假設我之前使用 1.16 版的 Operator（Postgres 15），即便把 Operator 更新到 1.21（預設為 Postgres 16 Operator），既有的叢集若沒有手動升級，仍會維持 PG 15。

此時若發生故障想嘗試還原，由於 Operator 版本是 1.21，會 Provisioning PG16，而原本 S3 中儲存的是 PG15 資料，可能就還原不了。

這種情況下，可以把 Operator 降版到當時安裝的版本，啟動相同版本的 PG 後還原 -> 更新；或者透過設定 Image 來使用相同主要版本。

**若可能，強烈建議先建立一個新叢集確認能正常還原後，再進行下一步！！**

### 4. 叢集升級

次要版本升級會自動進行，主要版本升級時按下面的步驟做，通常不會有什麼問題。（試過 15 -> 16）

線上升級請參考 [The Current State of Major PostgreSQL Upgrades with CloudNativePG](https://www.enterprisedb.com/blog/current-state-major-postgresql-upgrades-cloudnativepg-kubernetes)，

如果是離線升級（允許停機）：

1. 在既有叢集用 pg_dumpall 匯出資料
2. 建立新叢集
3. 將第 1 步匯出的資料倒入第 2 步
4. 把指向舊叢集的應用程式改為指向新叢集
5. 測試後刪除舊叢集

### 5. 結尾

我目前用 CNPG 已經穩定運作了約 7~8 個月。

考慮到我打掃時經常順手把線給拔掉(...)，在一般情況下它已經是足夠穩固的 Postgres DB，即使我不小心把整個叢集斷電，也能輕鬆從既有的 S3 還原資料，所以我覺得這套系統的可靠性相當高。

當然，如果預算充裕，使用託管 RDB 才是最好的選擇；不過作為技術驗證，希望大家也能知道有這樣的系統！

到這一步，伺服器等所需的系統應該都已備齊，可以開始開發了。就從簡單的開始一個個動手吧。

另外在 Kubernetes 內，可以透過 cnpg-lb-rw.cnpg-cluster 這個名稱存取該 DB。例如 `jdbc:postgresql://cnpg-lb-rw.cnpg-cluster:5432/my_app` 這樣。

讀這麼長的文章辛苦大家了！下次將介紹如何利用 nvidia-device-plugin 在 K3S 上使用 GPU！
