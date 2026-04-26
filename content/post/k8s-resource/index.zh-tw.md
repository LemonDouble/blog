---
title: "自用 K8S 資源整理"
slug: f13e7c85c56b2c7c
date: 2023-01-26T23:15:42+09:00
image: cover.png
math: 
categories:
    - K8S
tags:
    - K8S
    - Pod
    - Service
    - Deployment
    - 速查表
---

_註：筆者居住於韓國，部分內容包含韓國特有的背景。_

* `k get pod <podname> -o` 或 `k describe pod <podname>`：詳細查看狀態資訊
* `k logs -f <podname> -c <container-name>`：查看 log
* `k get pod <podname> --show-lables`：查看 Label 資訊

### 1. Pod

* 一個或多個 Container 的集合
* 執行的最小單位

* 特性
  * 若由多個 Container 組成,則永遠被指派到同一個 Node
  * 配發獨立的 Pod IP（每個 Pod 一個）
  * Pod 內的 Container 共用網路,因此可以透過 Localhost 連線
  * 共用 Volume

* Pod 可能擁有的 State
  * `Pending`：建立命令已傳達到 Master Node,但尚未建立的狀態
  * `ContainerCreating`：已指派到特定 Node 並正在建立中（映像檔下載等）
  * `Running`：正常執行中
  * `Completed`：一次性執行後結束的 Pod（批次作業等）作業完成的情況
  * `Error`：Pod 出現問題的情況
  * `CrashLoopBackOff`：持續發生錯誤、Crash 反覆出現的狀態

* spec.restartPolicy
  * `Always`：結束時永遠重新啟動（default）
  * `Never`：不重新啟動
  * `OnFailure`：僅在失敗時重新啟動（正常結束時不會重新啟動）

* 狀態檢查
  * `livenessProbe`：檢查 Pod 是否正常運作
  * `readinessProbe`：在 Pod 建立後,檢查是否已準備好接收流量（可透過 READY 0/1 狀態查看）

### 2. Service

* 把它想成 Reverse Proxy 就容易理解。Pod 並不穩定,因此 Service 站在它前面提供穩定的 Endpoint
* 類似 DNS,可以透過 Service 名稱連線。若建立了名為 myservice 的 service,就可以用 myservice 這個名稱連線

* Service 的 DNS
  * `<ServiceName>.<Namespace>.svc.cluster.local`
  * 範例：default Namespace 中的 myservice -> `myservice.default.svc.cluster.local`
  * svc.cluster.local 是 K8S 的預設值

* Service 是如何擁有 DNS 的呢?
  * 透過 `k exec client -- cat /etc/resolv.conf` 查看 Pod 的 DNS 設定 -> 可以查到 Nameserver 的 IP
  * 輸入 `k get svc -n kube-system` 後看到的 CoreDNS 就是這個 IP
  * 透過 CoreDNS 這個服務來使用 DNS

* Service 的種類
  * `ClusterIP`：（default）只能從 K8S 叢集內部存取。
  * `NodePort`：把 Localhost 的特定 Port 與 Service 的特定 port 連接（聯想 Docker）
    * 不過 Nodeport 會把所有 Node 的指定 port 全部連到該 Service
    * 例如有 32000 的 Nodeport,那 `master:32000`、`worker01:32000` 全部都會連到 32000 port
  * `LoadBalancer`：建立 Load Balancer(由 Cloud 提供,或 Software)並將流量分散到 Pod
    * 外部只要知道 LoadBalancer 的 IP 即可,非常方便。(ClusterIP -> Pod 的穩定服務 Endpoint,LoadBalancer -> Node 的穩定服務 Endpoint)
    * 透過 External-IP 即可從外部存取
  * `ExternalName`：想在 K8S 網路內使用外部服務時
    * 例：把 google.com 設定為 ExternalName 為 google-svc,就可以用 `curl google-svc` 取得 google.com 的資料

### 3. Controller

* 比較 Current State 和 Desired State,反覆執行 Task 使其一致

* `ReplicaSet`
  * 複製 Pod 以維持一定數量
  * 可以用 spec.replicas: 2 這樣的方式宣告要複製的 Pod 數量
  * 透過 selector.matchLables 選擇 label 匹配的 Pod

* `Deployment`

  * 利用 ReplicaSet 建立一定數量的 Pod,並在此基礎上加入部署功能（Deployment -> ReplicaSet -> Pod）
  * 支援 Rolling Update,可調整更新中的 Pod 比例
    * 透過 `spec.strategy.type: RollingUpdate` 設定 Update 策略
    * `spec.strategy.rollingUpdate.maxUnavailable: 25%`：（小數點以下無條件捨去）指定部署期間最多可中斷的 Pod 百分比。10 個的 25% 就是 2 個 Pod
    * `spec.strategy.rollingUpdate.maxSurge: 25%`：（小數點以下無條件進位）指定部署期間最多可超出的 Pod 百分比。10 個的 25% 就是 3 個 Pod
  * 儲存 Update History 並支援 Rollback
    * `k rollout undo deployment <Deployment名稱>`

* StatefulSet
  * 每個 Pod 都有唯一的識別碼
  * 每個 Pod 可以將獨有的資料保存在儲存空間中（例如:Database）
  * 保證 Pod 建立的順序 -> 用於對順序敏感的應用程式（第一個應用程式為 Master,第二個以後為 Slave 等）
  * 以 `name-<序號>`（從 0 開始）的形式為 Pod 命名
  * 刪除時依倒序刪除（從編號大的開始刪除）

* DaemonSet
  * 想在所有 Node 上執行相同 Pod 時使用 -> 監控、log 收集等

### 4. Job

* `Job`
  * 一次性執行後結束的程式（ML 訓練、Batch Job 等）
  * 可以用 `spec.backoffLimit: 2` 這樣的方式設定失敗時的重試策略。（重試 2 次）

* CronJob
  * 提供類似 Crontab 的功能,可以週期性地執行 Job
  * 用 `spec.schedule: "*/1 * * * *"` 這樣的方式指定

### Namespace

* 用來在邏輯上劃分叢集


* default：預設 Namespace
* kube-system：存放 K8S 核心元件(DNS、網路等)的 Pod
* kube-public：存放可對外公開資源的 Namespace
* kube-node-lease：用來讓 Node 向 master 通知自己存活?的 namespace

### ConfigMap, Secret

* `ConfigMap`
  * 儲存 Metadata（設定值）
  * 以 Key-value 形式儲存值

  * 可按下列方式用 Yaml 宣告
  ```yaml
  # game-volume.yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: game-volume
  spec:
    restartPolicy: OnFailure
    containers:
    - name: game-volume
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh", "-c", "cat /etc/config/game.properties" ]
      volumeMounts:
      - name: game-volume
        mountPath: /etc/config
    volumes:
    - name: game-volume
      configMap:
        name: game-config
  ```

  * 之後可以在其他 Pod 中透過 Volume 掛載或環境變數使用
    * Volume：
  ```yaml
  .....
      volumeMounts:
    - name: game-volume
      mountPath: /etc/config
  volumes:
  - name: game-volume
    configMap:
      name: game-config
  ```
  * Env：
  ```yaml
  .....
    envFrom:
    - configMapRef:
        name: monster-config
  ```

* Secrets
  * 使用 tmpfs（一種讓記憶體可以像檔案一樣處理的系統）,在安全性上有優勢
  * 查詢時進行 Base64 編碼（不是加密,只是做一次編碼）

### Appendix 1. 僅在特定 Node 上執行 Pod

* 範例:區分 SSD/HDD Node 進行配置的方法

* 為每個 Node 加上 label
  * `kubectl label node master disktype=ssd`
  * `kubectl label node worker disktype=hdd`

* 接著透過 nodeSelector 選擇 label
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-selector
spec:
  containers: 
  - name: nginx
    image: nginx
  # 選擇特定的節點標籤
  nodeSelector:
    disktype: ssd
```
