---
title: "自用 K8S 资源整理"
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

_注：作者居住在韩国，部分内容包含韩国特有的背景。_

* `k get pod <podname> -o` 或 `k describe pod <podname>`：详细查看状态信息
* `k logs -f <podname> -c <container-name>`：查看日志
* `k get pod <podname> --show-lables`：查看 Label 信息

### 1. Pod

* 一个或多个 Container 的集合
* 执行的最小单位

* 特点
  * 如果由多个 Container 组成，始终被分配到同一个 Node
  * 分配独立的 Pod IP（每个 Pod 一个）
  * Pod 内的 Container 共享网络，因此可以通过 Localhost 访问
  * 共享 Volume

* Pod 可能拥有的 State
  * `Pending`：创建命令已传达到 Master Node，但尚未创建的状态
  * `ContainerCreating`：已分配到特定 Node 并正在创建中（镜像下载等）
  * `Running`：正常运行中
  * `Completed`：一次性执行后结束的 Pod（批处理作业等）作业完成的情况
  * `Error`：Pod 出现问题的情况
  * `CrashLoopBackOff`：持续发生错误、Crash 反复出现的状态

* spec.restartPolicy
  * `Always`：终止时始终重启（default）
  * `Never`：不重启
  * `OnFailure`：仅在失败时重启（正常终止时不会重启）

* 状态检查
  * `livenessProbe`：检查 Pod 是否正常运行
  * `readinessProbe`：在 Pod 创建后,检查是否已准备好接收流量（可通过 READY 0/1 状态查看）

### 2. Service

* 把它想象成 Reverse Proxy 就容易理解。Pod 不稳定，所以由 Service 站在它前面提供稳定的 Endpoint
* 类似 DNS，可以通过 Service 名称连接。如果创建了名为 myservice 的 service，就可以通过 myservice 这个名字连接

* Service 的 DNS
  * `<ServiceName>.<Namespace>.svc.cluster.local`
  * 示例：default Namespace 中的 myservice -> `myservice.default.svc.cluster.local`
  * svc.cluster.local 是 K8S 的默认值

* Service 是如何拥有 DNS 的？
  * 通过 `k exec client -- cat /etc/resolv.conf` 查看 Pod 的 DNS 设置 -> 可以查到 Nameserver 的 IP
  * 输入 `k get svc -n kube-system` 后看到的 CoreDNS 就是这个 IP
  * 通过 CoreDNS 这个服务来使用 DNS

* Service 的种类
  * `ClusterIP`：（default）只能在 K8S 集群内部访问。
  * `NodePort`：把 Localhost 的特定 Port 与 Service 的特定端口连接（联想 Docker）
    * 不过 Nodeport 会把所有 Node 的指定端口都连接到该 Service
    * 例如有 32000 的 Nodeport，那么 `master:32000`、`worker01:32000` 全部连接到 32000 端口
  * `LoadBalancer`：创建 Load Balancer（Cloud 提供，或 Software）并将流量分发到 Pod
    * 外部只需知道 LoadBalancer 的 IP 即可，很方便。（ClusterIP -> Pod 的稳定服务 Endpoint，LoadBalancer -> Node 的稳定服务 Endpoint）
    * 通过 External-IP 可以从外部访问
  * `ExternalName`：当想在 K8S 网络内使用外部服务时
    * 例：把 google.com 设置为 ExternalName 为 google-svc，就可以通过 `curl google-svc` 获取 google.com 的数据

### 3. Controller

* 比较 Current State 和 Desired State，反复执行 Task 使其一致

* `ReplicaSet`
  * 复制 Pod 以维持一定数量
  * 可以用 spec.replicas: 2 这样的方式声明要复制的 Pod 数量
  * 通过 selector.matchLables 选择 label 匹配的 Pod

* `Deployment`

  * 利用 ReplicaSet 创建一定数量的 Pod，并在此基础上添加部署功能（Deployment -> ReplicaSet -> Pod）
  * 支持 Rolling Update，可调整更新中的 Pod 比例
    * 通过 `spec.strategy.type: RollingUpdate` 设置 Update 策略
    * `spec.strategy.rollingUpdate.maxUnavailable: 25%`：（向下取整）指定部署期间最大可中断的 Pod 百分比。10 个的 25% 就是 2 个 Pod
    * `spec.strategy.rollingUpdate.maxSurge: 25%`：（向上取整）指定部署期间最大可超出的 Pod 百分比。10 个的 25% 就是 3 个 Pod
  * 保存 Update History 并支持 Rollback
    * `k rollout undo deployment <Deployment名称>`

* StatefulSet
  * 每个 Pod 都有唯一标识符
  * 每个 Pod 可以将独有的数据保存在存储中（例如：Database）
  * 保证 Pod 创建的顺序 -> 用于对顺序敏感的应用（第一个应用是 Master，第二个之后是 Slave 等）
  * 以 `name-<序号>`（从 0 开始）的形式给 Pod 命名
  * 删除时按倒序删除（从编号大的开始删除）

* DaemonSet
  * 想在所有 Node 上运行同一个 Pod 时使用 -> 监控、日志收集等

### 4. Job

* `Job`
  * 一次性执行后结束的程序（ML 训练、Batch Job 等）
  * 可以用 `spec.backoffLimit: 2` 这样的方式设置失败时的重试策略。（重试 2 次）

* CronJob
  * 提供类似 Crontab 的功能，可以周期性地执行 Job
  * 用 `spec.schedule: "*/1 * * * *"` 这样的方式指定

### Namespace

* 用于在逻辑上划分集群


* default：默认 Namespace
* kube-system：存放 K8S 核心组件（DNS、网络等）的 Pod
* kube-public：存放可对外公开资源的 Namespace
* kube-node-lease：用于 Node 向 master 通知自己存活？的 namespace

### ConfigMap, Secret

* `ConfigMap`
  * 保存 Metadata（配置值）
  * 以 Key-value 形式保存值

  * 可按如下方式用 Yaml 声明
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

  * 之后可以在其他 Pod 中通过 Volume 挂载或环境变量使用
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
  * 使用 tmpfs（一种把内存当作文件来处理的系统），在安全性上有优势
  * 查询时进行 Base64 编码（不是加密，只是做了一次编码）

### Appendix 1. 仅在特定 Node 上运行 Pod

* 示例：区分 SSD/HDD Node 进行调度的方法

* 给每个 Node 打 label
  * `kubectl label node master disktype=ssd`
  * `kubectl label node worker disktype=hdd`

* 之后通过 nodeSelector 选择 label
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-selector
spec:
  containers: 
  - name: nginx
    image: nginx
  # 选择特定的节点标签
  nodeSelector:
    disktype: ssd
```
