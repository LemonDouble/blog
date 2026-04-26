---
title: "在家用 Raspberry Pi 叢集打造資料中心 - 5. ArgoCD GitOps 設定與 Longhorn 安裝"
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
    - 儲存
---

_註：筆者居住於韓國，部分內容包含韓國特有的背景。_

### 1. 開始 GitOps

ArgoCD 已經安裝完成，接下來整理一下儲存庫的結構。

我們會使用 App On Apps 模式來打造一套簡潔的部署方式。

就算現在還不太理解，先跟著做就好。

把上次建立的空儲存庫整理成下面這樣的結構。

```sh
/apps
    ㄴ/disabled
    ㄴ/enabled
/init
/modules
```

接著，把先前操作過的 ip_address_pool.yaml、(ArgoCD 的) ingress.yaml、traefik-update.yaml、values.yaml 全部放進 init 資料夾。

把最初設定時的檔案保存在 Git，是為了在日後需要還原叢集時，可以用 `kubectl -f 檔案名稱` 快速重新套用這些設定檔。

像下面這樣加入檔案，再寫一份 README.md 紀錄到目前為止做了哪些事，叢集出狀況時對還原會有很大的幫助。README.md 請自己寫！

```sh
/init
    ㄴ ip_address_pool.yaml
    ㄴ ingress.yaml
    ㄴ traefik-update.yaml
    ㄴ values.yaml
```

接著，在 init 資料夾中參考下方的 yaml 加入 root-app.yaml。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <隨意取一個喜歡的名字 (筆者是 lemon-k8s-apps)>
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

逐項拆解：

1. 新增一個由 ArgoCD 管理的應用程式
2. 該應用程式參考 GitHub 上自己的儲存庫
3. Path 會執行上面設定的 apps/enabled 資料夾下所有 yaml 檔

然後執行 `kubectl apply -f root-app.yaml` 安裝 root app。

之後回到 ArgoCD，確認自己的第一個應用程式是否正確顯示。如果 Sync 失敗並提示儲存庫中沒有該 Path，那就是已經正常連線了。

### 2. 安裝 Longhorn

* 假設除了主 USB(裝有作業系統的 USB)之外，至少還插了一個用於儲存的 USB、SSD 或 HDD。

**截至 2023 年 12 月，使用 ArgoCD 首次安裝時，Longhorn 的初始 Batch Job 會因為一個 bug 而失敗，所以我們先手動安裝一次再轉移到 ArgoCD。**

Longhorn 是分散式儲存管理員，提供以下功能。(懶得讀的話，可以簡單把它當成提供類似 S3 的功能！)

1. **分散式儲存**

依照設定，將檔案分散保存到 N 個儲存裝置上。

如果某次故障導致一個儲存裝置損毀，原本放在它上面的資料會自動遷移到其他儲存裝置，確保至少維持 N 份備援。

如此一來，即使一顆 SSD 故障，其他儲存裝置上仍有相同資料，程式與系統就能繼續正常運作。

2. **Storage Tiering**

有些系統是 I/O 非常重的工作負載，需要高效能儲存；

有些系統則是不常讀取的檔案非常多，比起效能更需要容量。

Longhorn 支援 StorageClass，可以據此把儲存分層使用。

例如：在 I/O 重要的情況下，可分配只由 NVMe SSD 組成的 storage class；如果只是大量檔案儲存，則分配只由 HDD 組成的 storage class。

那麼！透過下列指令安裝 Longhorn。

```sh
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace --set service.ui.loadBalancerIP="192.168.0.201" --set service.ui.type="LoadBalancer"
```

如果安裝成功，從瀏覽器開啟 `http://192.168.0.201`，確認 Longhorn UI 有正常顯示。

### 3-1. 使用 ArgoCD 安裝 - 加入設定檔

既然手動安裝已經完成一次，我們就把已經安裝好的 Longhorn 移轉到 ArgoCD。

資料夾結構與部署策略如下。依下方結構，部署 root-app 後底下的應用程式會自動啟動。

![資料夾結構，Root App 會部署 enabled 資料夾，enabled 資料夾下的 yaml 檔會部署 modules/檔名](image.png)

這一步同樣先照做。加入下面的檔案。

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
加入一個自動部署 modules/longhorn-system 的應用程式。

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

透過 `helm.parameters`，可以像上面那樣加入要寫進 values.yaml 的值。


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
加入 ssd、hdd 儲存。如果你使用 USB，也可以把名稱改成 longhorn-usb 等。

`numberOfReplicas`：副本的數量。筆者的情況，SSD 用來存放應用程式需要的資料，為了穩定性至少設成 2 份；HDD 只是單純的檔案儲存，因此設成 1 份。
`fsType`：檔案系統類型。預計使用 ext4。如果跟著本指南操作，維持一樣即可。
`diskSelector`：稍後註冊磁碟時會用到，請記下來。

作業完成後，Git commit 之後 Push！

### 3-2. 使用 ArgoCD 安裝 - 實際進行部署

接下來進入最初設定的 ArgoCD，親自進行部署。

![ArgoCD Root App 截圖](image-1.png)

我們需要知道的是 `Sync`、`Refresh` 按鈕。

- `Sync`：從 Git 取得新的叢集狀態後，將實際的叢集狀態與 Git 同步。
- `Refresh`：從 Git 取得新的叢集狀態，並不會同步叢集。

因此，按下 Sync 後 ArgoCD 就會開始部署。

接著依序進入 Longhorn-system -> Longhorn，依次按下 Sync 來進行部署。

### 結語

辛苦了！我們終於把之前散落各處的 YAML 檔整理完，往 GitOps 邁出了第一步。

下次將透過 Longhorn 實際註冊 SSD 與 HDD。
