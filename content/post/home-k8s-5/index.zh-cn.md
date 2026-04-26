---
title: "在家用 Raspberry Pi 集群搭建数据中心 - 5. ArgoCD GitOps 配置与 Longhorn 安装"
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
    - 存储
---

_注：作者居住在韩国，部分内容包含韩国特有的背景。_

### 1. 开始 GitOps

ArgoCD 已经安装好了，接下来整理一下仓库结构。

我们会使用 App On Apps 模式来构建一种简洁的部署方式。

即使现在不太理解，先跟着做就行。

把上次创建的空仓库整理成下面这样的结构：

```sh
/apps
    ㄴ/disabled
    ㄴ/enabled
/init
/modules
```

然后，把之前操作过的 ip_address_pool.yaml、(ArgoCD 的) ingress.yaml、traefik-update.yaml、values.yaml 全部加到 init 文件夹里。

把最初设置时的文件保存在 Git 里，是为了在以后需要恢复集群时，能够通过 `kubectl -f 文件名` 快速重新应用这些配置文件。

像下面这样添加文件，再写一份 README.md 记录到目前为止做了哪些工作，集群出问题时对恢复非常有帮助。README.md 请自己写！

```sh
/init
    ㄴ ip_address_pool.yaml
    ㄴ ingress.yaml
    ㄴ traefik-update.yaml
    ㄴ values.yaml
```

接着，在 init 文件夹中参考下面的 yaml 添加 root-app.yaml。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <随便起一个喜欢的名字 (我这里是 lemon-k8s-apps)>
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

逐项拆解：

1. 添加一个由 ArgoCD 管理的应用
2. 该应用引用 GitHub 上的我自己的仓库
3. Path 会执行上面配置的 apps/enabled 中所有 yaml 文件

然后执行 `kubectl apply -f root-app.yaml` 安装 root app。

之后回到 ArgoCD，确认我的第一个应用是否正常显示。如果 Sync 失败，并提示仓库中没有该 Path，说明已经正常连接了。

### 2. 安装 Longhorn

* 假设除了主 USB(装有操作系统的 USB)之外，至少还插了一块用于存储的 USB、SSD 或 HDD。

**截至 2023 年 12 月，使用 ArgoCD 首次安装时，Longhorn 的初始 Batch Job 会因为一个 bug 失败，所以我们先手动安装一次，再迁移到 ArgoCD。**

Longhorn 是一个分布式存储管理器，提供以下功能。(懒得读的话，可以简单理解为它提供了类似 S3 的功能！)

1. **分布式存储**

按照配置，将文件分散保存到 N 个存储设备上。

如果某次故障导致一台存储被破坏，原先存放在它上面的数据会自动迁移到其他存储，确保至少保持 N 个副本。

这样即使一块 SSD 挂掉，其他存储上仍有相同的数据，程序和系统就能继续正常运行。

2. **Storage Tiering**

有些系统是 I/O 非常重的工作负载，需要很高的存储性能；

而有些系统则是不常读取的文件特别多，比起性能更需要容量。

Longhorn 支持 StorageClass，可以据此把存储分层使用。

例如，对 I/O 要求高的场景，可以分配只由 NVMe SSD 组成的 storage class；对单纯需要大量文件存储的场景，则分配只由 HDD 组成的 storage class。

那么！通过下面的命令安装 Longhorn。

```sh
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace --set service.ui.loadBalancerIP="192.168.0.201" --set service.ui.type="LoadBalancer"
```

如果安装成功，在浏览器中访问 `http://192.168.0.201`，确认能看到 Longhorn UI。

### 3-1. 用 ArgoCD 来安装 - 添加配置文件

既然手动安装已经完成一次，我们就把已经安装好的 Longhorn 迁移到 ArgoCD。

文件夹结构和部署策略如下。按照下面的结构，部署 root-app 后，下层应用会自动启动。

![文件夹结构，Root App 部署 enabled 文件夹，enabled 文件夹下的 yaml 文件部署 modules/文件名](image.png)

这一步同样先照做。添加下面的文件。

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
添加一个自动部署 modules/longhorn-system 的应用。

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

通过 `helm.parameters`，可以像上面这样添加要写入 values.yaml 的值。


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
添加 ssd、hdd 存储。如果你用的是 USB，也可以把名称改成 longhorn-usb 等。

`numberOfReplicas`：副本数量。我这里 SSD 用于存放应用所需的数据，为了稳定性至少设为 2 个；HDD 只是单纯文件存储，所以设为 1 个。
`fsType`：文件系统类型。这里使用 ext4。如果你按照本指南操作，保持一致即可。
`diskSelector`：稍后注册磁盘时会用到。请记住它。

完成后，提交 Git 然后 Push！

### 3-2. 用 ArgoCD 来安装 - 实际部署

现在进入最初配置的 ArgoCD，亲自进行部署。

![ArgoCD Root App 截图](image-1.png)

我们要关注的是 `Sync`、`Refresh` 按钮。

- `Sync`：从 Git 拉取新的集群状态，然后将实际集群状态与 Git 同步。
- `Refresh`：从 Git 拉取新的集群状态，但不会同步集群。

所以点击 Sync 后，ArgoCD 就会开始部署。

接着按 Longhorn-system -> Longhorn 的顺序逐层进入，依次点击 Sync 进行部署。

### 结束语

辛苦了！终于把之前散落各处的 YAML 文件整理好，向 GitOps 迈出了第一步。

下次我们将通过 Longhorn 真正注册 SSD 和 HDD。
