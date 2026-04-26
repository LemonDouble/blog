---
title: "自分用にまとめるK8Sリソースまとめ"
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
    - チートシート
---

_注：筆者は韓国在住のため、本文には韓国特有の文脈が含まれることがあります。_

* `k get pod <podname> -o` または `k describe pod <podname>`：状態情報を詳細に確認
* `k logs -f <podname> -c <container-name>`：ログを確認
* `k get pod <podname> --show-lables`：Label情報を確認

### 1. Pod

* 1つ以上のContainerの集合
* 実行の最小単位

* 特徴
  * 複数のContainerで構成されている場合、常に同一Nodeに割り当てられる
  * 固有のPod IPが割り当てられる（Podごとに1つ）
  * Pod内のContainerはネットワークを共有するため、Localhostでアクセス可能
  * Volumeを共有

* Podが取り得るState
  * `Pending`：Master Nodeに生成命令は伝わったが、まだ生成されていない状態
  * `ContainerCreating`：特定のNodeに割り当てられて生成中（イメージのダウンロードなど）
  * `Running`：正常に実行中
  * `Completed`：一度実行されて終了するPod（バッチ処理など）の作業が完了した場合
  * `Error`：Podに問題が発生した場合
  * `CrashLoopBackOff`：エラーが継続的に発生し、Crashが繰り返されている状態

* spec.restartPolicy
  * `Always`：終了時に常に再起動（default）
  * `Never`：再起動しない
  * `OnFailure`：失敗した場合のみ再起動（正常終了時は再起動しない）

* 状態確認
  * `livenessProbe`：Podが正常に動作しているかを確認
  * `readinessProbe`：Podが生成された後、トラフィックを受け取る準備が完了したかを確認（READY 0/1 の状態で確認可能）

### 2. Service

* Reverse Proxyだと思えば分かりやすいです。Podは不安定ですが、その前に立って安定したEndpointを提供します
* DNSのようにService名で接続可能。myserviceというserviceを作成した場合、myserviceという名前で接続可能

* ServiceのDNS
  * `<ServiceName>.<Namespace>.svc.cluster.local`
  * 例：default Namespaceのmyservice -> `myservice.default.svc.cluster.local`
  * svc.cluster.localはK8Sのデフォルト値

* ServiceはどのようにDNSを持つのか？
  * `k exec client -- cat /etc/resolv.conf` でPodのDNS設定を確認 -> NameserverのIPが分かる
  * `k get svc -n kube-system` を入力すると表示されるCoreDNSがそのIP
  * CoreDNSというサービスを通じてDNSを利用可能

* Serviceの種類
  * `ClusterIP`：（default）K8Sクラスタ内部からのみアクセス可能。
  * `NodePort`：Localhostの特定のPortをServiceの特定のポートと接続（Dockerをイメージ）
    * ただし、NodeportはすべてのNodeの特定のポートを当該Serviceに接続する
    * 例えば32000番のNodeportがあれば、`master:32000`、`worker01:32000` 全てが32000番ポートに接続される
  * `LoadBalancer`：Load Balancerを生成して（Cloud提供、もしくはSoftware）Podに分散
    * 外部からはLoadBalancerのIPだけ知っていれば良いので便利です。（ClusterIP -> Podの安定したサービスEndpoint、LoadBalancer -> Nodeの安定したサービスEndpoint）
    * External-IPを通じて外部からアクセス可能
  * `ExternalName`：外部のサービスをK8Sネットワーク内で使いたいとき
    * 例：google.comをExternalNameとしてgoogle-svcに設定すると、`curl google-svc` でgoogle.comのデータを取得できる

### 3. Controller

* Current StateとDesired Stateを比較し、一致するようにTaskを繰り返す

* `ReplicaSet`
  * Podを複製して一定の個数を維持する
  * spec.replicas: 2 のように複製するPodの個数を宣言可能
  * selector.matchLables を通じてlabelがMatchするPodを選択可能

* `Deployment`

  * ReplicaSetを使って一定数のPodを生成し、そこにデプロイ機能を追加（Deployment -> ReplicaSet -> Pod）
  * Rolling Updateをサポートし、更新されるPodの比率を調整可能
    * `spec.strategy.type: RollingUpdate` でUpdate戦略を設定
    * `spec.strategy.rollingUpdate.maxUnavailable: 25%`：（小数点以下切り捨て）デプロイ中に最大停止可能なPodの%を指定。10個に対して25%なら2個のPod
    * `spec.strategy.rollingUpdate.maxSurge: 25%`：（小数点以下切り上げ）デプロイ中に最大超過可能なPodの%を指定。10個に対して25%なら3個のPod
  * Update Historyを保存し、Rollback可能
    * `k rollout undo deployment <Deployment名>`

* StatefulSet
  * Podごとに固有の識別子が存在
  * Podごとに固有のデータをストレージに保存可能（例：Database）
  * Pod生成の順序を保証 -> 順序に敏感なアプリケーション（最初のアプリケーションはMaster、2番目以降はSlaveなど）に使用
  * `name-<連番>`（0から開始）の形式でPod名を付与
  * 削除される場合は逆順削除（番号が大きいものから削除）

* DaemonSet
  * すべてのNodeに同じPodを実行させたいとき -> モニタリング、ログ収集など

### 4. Job

* `Job`
  * 単発で実行されて終了するプログラム（ML学習、Batch Jobなど）
  * `spec.backoffLimit: 2` のように失敗時の再試行ポリシーを設定可能（2回再試行）

* CronJob
  * 周期的にJobを実行できるよう、Crontabのような機能を提供
  * `spec.schedule: "*/1 * * * *"` のように指定

### Namespace

* クラスタを論理的に分割するために使用


* default：基本Namespace
* kube-system：K8Sのコアコンポーネント（DNS、ネットワークなど）のPodが存在する
* kube-public：外部に公開可能なリソースを置くNamespace
* kube-node-lease：Nodeが生きていることをmasterに知らせる用途？のnamespace

### ConfigMap, Secret

* `ConfigMap`
  * Metadata（設定値）を保存
  * Key-value形式で値を保存

  * 次のようにYamlで宣言可能
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

  * その後、他のPodからVolumeで接続したり、環境変数として使用可能
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
  * tmpfs（メモリをファイルのように扱えるようにするシステム）を使ってセキュリティ上のメリットがある
  * 参照時はBase64エンコーディング（暗号化ではなく、単に一度エンコードしているだけ）

### Appendix 1. 特定のNodeでのみPodを実行する

* 例：SSD/HDD Nodeを区別して配置する方法

* 各Nodeにlabelを付与
  * `kubectl label node master disktype=ssd`
  * `kubectl label node worker disktype=hdd`

* その後、nodeSelectorでlabelを選択
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-selector
spec:
  containers: 
  - name: nginx
    image: nginx
  # 特定のノードラベルを選択
  nodeSelector:
    disktype: ssd
```
