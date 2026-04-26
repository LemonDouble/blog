---
title: "在家用树莓派集群搭建数据中心 - 9. 使用 CloudNativePG 在 Kubernetes 上运维数据库"
slug: a4ded8cced5f045c
date: 2023-12-12T22:09:10+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - 树莓派
tags:
    - K8S
    - K3S
    - 树莓派
    - CloudNativePG
    - PostgreSQL
    - Operator
    - HA
---

_注：作者居住在韩国，部分内容包含韩国特有的背景。_

### 序言

数据库管理起来很棘手。

与其他 Pod 不同，需要操心数据保管、备份、管理，还要关注 Failover 和性能。

所以据我所知，即便其他工作负载都在 Kubernetes 集群上运行，单独把 DB 放在 AWS RDS 之类的托管系统或独立实例上的情况也很常见。

但这真的重要吗？要在家里搭建数据中心，难道不应该有一个托管 DBMS 吗？

那就来做一个吧。我们将利用 Kubernetes 的 Operator Pattern 通过 [CloudNativePG](https://cloudnative-pg.io/) 部署一个 1 主 2 从的集群，并配置为可从内网访问。

虽然不是 Postgres Operator，但读一读 [用 Mysql Operator 在 Kubernetes 环境运维 Mysql DB](https://nangman14.tistory.com/79#5.%20Mysql%20Operator%20Failover%20%ED%85%8C%EC%8A%A4%ED%8A%B8-1)（韩文）这篇文章，对跟上节奏可能会有些帮助。

### 2. 安装

安装分两步进行。

1. 安装 CNPG Operator
2. 安装 CNPG Cluster

Operator 的作用是监视 Cluster 是否保持正常状态。
实际使用的数据库集群通过第 2 步安装。

我们一步一步开始吧！这次也使用 ArgoCD 快速部署。

**1. 安装 CNPG Operator**

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

很简单吧？安装后部署一下，把 CNPG Operator 装上。

**2. 安装 CNPG Cluster**

我们要搭建的集群形态如下：

1. 每天 UTC 0 点（KST 上午 9 点）执行 S3 Daily Backup。
2. 总共由 3 个 Pod 组成。每个 Pod 分布在多个节点上，以预防可能的不幸（？）。
3. 通过 192.168.0.x 的 IP 可从内网访问 DB。

我们一个一个开始吧！

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

为了管理方便，开启了 Superuser 访问，
DB 容量设为 10GB。（之后可以扩展。）

此外，为了在万一发生意外事故时(...)能够轻松备份，将备份存储设置为保存到 AWS S3，并保留最长 90 天。

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

简单的 Daily Backup 资源。

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

我这里允许通过 192.168.0.206 地址访问。
之后查询或管理数据库时，可以从内网通过 `192.168.0.206` 访问进行管理。

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

在 AWS 上授予 S3FullAccess 或特定 Bucket 的访问权限后，签发 ACCESS KEY 和 Secret，再用之前配置好的 Sealed Secret 注册。

`modules/cnpg-cluster/sealed-superuser-secrets.yaml`

生成过程稍微有点复杂！

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: superuser-secrets
  namespace: cnpg-cluster
type: kubernetes.io/basic-auth
stringData:
  username: <我要使用的 ID，不要 b64 编码，原样填写>
  password: <我要使用的密码，不要 b64 编码，原样填写>
```

先按上面的内容以 secret.yaml 为名创建 Secret，然后用

```sh
cat secret.yaml | kubeseal --controller-namespace=sealed-secrets-system --controller-name=sealed-secrets -oyaml > sealed-superuser-secrets.yaml
```

将其转换为 Sealed Secret，然后使用 Sealed Secret！

之后等待 Provisioning（需要一些时间），就可以通过 192.168.0.206 用刚才设置的 ID/Password 登录使用 DB 了！

**如果安装正常，请务必！设个第二天的提醒，确认 S3 对应文件夹中是否正常备份了！！！**

### 3. 恢复

过了一天确认正常备份后，**请务必！确认能否恢复**。
等数据丢了再确认就太晚了……


分享一下我的恢复配置文件。

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

按上面的方式给新集群加上 bootstrap、externalClusters 选项后，集群首次启动时会自动用已有的 S3 文件恢复数据。

连接/确认无误，恢复结束后，把 bootstrap、externalClusters 选项删除，再加回原来的备份选项即可。

这里需要注意的是，
如果 Postgres 的主版本不同，恢复可能无法正常进行。

举个例子，假设我之前用的是 1.16 版本的 Operator（Postgres 15），即使把 Operator 升级到 1.21（默认是 Postgres 16 Operator），已生成的集群在没有手动升级的情况下仍会保持 PG 15。

此时如果发生故障要尝试恢复，由于 Operator 版本是 1.21，会去 Provisioning PG16，而原本 S3 中保存的是 PG15 的数据，恢复就可能失败。

遇到这种情况，可以把 Operator 降级到当时安装的版本，启动相同版本的 PG，然后恢复 -> 升级；或者通过设置 Image 来使用相同的主版本。

**如果可以，强烈建议先创建一个新集群确认能正常恢复后，再进行下一步！！**

### 4. 集群升级

小版本升级会自动进行，主版本升级时按下面的方式做，通常不会有什么问题。（试过 15 -> 16）

在线升级请参考 [The Current State of Major PostgreSQL Upgrades with CloudNativePG](https://www.enterprisedb.com/blog/current-state-major-postgresql-upgrades-cloudnativepg-kubernetes)，

如果是离线升级（允许停机）：

1. 在原集群用 pg_dumpall 导出数据
2. 创建新集群
3. 把第 1 步导出的数据灌入第 2 步
4. 把指向原集群的应用改为指向新集群
5. 测试后删除原集群

### 5. 结尾

我目前用 CNPG 已经稳定运行了大约 7~8 个月。

考虑到我打扫卫生时经常顺手把线给拔了(...)，在一般情况下它足以作为相当稳健的 Postgres DB 来使用，即便我不小心把整个集群断电，也能简单地从已有的 S3 恢复数据，所以我觉得它的可靠性相当高。

当然，如果预算充裕，使用托管 RDB 才是最好的选择，但作为技术验证，希望大家也知道有这样一种系统！

到这一步，服务器等所需的系统应该都已就绪，可以开始开发了。我们就从简单的开始一步步搭吧。

另外在 Kubernetes 内，可以通过 cnpg-lb-rw.cnpg-cluster 这个名字访问该 DB。例如 `jdbc:postgresql://cnpg-lb-rw.cnpg-cluster:5432/my_app` 这样。

读这么长的文章辛苦大家了！下次将介绍如何利用 nvidia-device-plugin 在 K3S 上使用 GPU！
