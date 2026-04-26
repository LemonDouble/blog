---
title: "自宅でラズベリーパイクラスターでデータセンターを構築する - 8-1. Docker Registry UIの設定とRegistry GCの使い方"
slug: a68cc3e1f5e8a95b
date: 2023-12-10T11:39:46+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - ラズベリーパイ
tags:
    - K8S
    - K3S
    - ラズベリーパイ
    - Docker Registry
    - GC
    - Web UI
---

_注：筆者は韓国在住のため、本文には韓国特有の文脈が含まれることがあります。_

### 1. Private Registryは便利だけど、管理がとても面倒です！

CLIが大好きな方もいらっしゃいますが、私はコマンドを覚えるのが本当に苦手なので、GUIと記録を好みます。

Docker RegistryもCLIで管理できますが、もう少し楽にするために [docker-registry-browser](https://github.com/klausmeyer/docker-registry-browser) を使います。

このような感じのGUIを使う予定です。

![Alt text](image.png)

![Alt text](image-1.png)

### 2. インストール

既存のdocker-registry-systemに一緒にインストールします。

既存ファイルの下に `---` を追加してから記述してください。

`modules/docker-registry-system/deployment.yaml`

- 既存のdeployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: docker-registry-ui-deployment
  namespace: docker-registry-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: docker-registry-ui-app
  template:
    metadata:
      labels:
        app: docker-registry-ui-app
    spec:
      nodeSelector:
        node-type: worker
      containers:
        - name: docker-registry-ui-pod
          image: klausmeyer/docker-registry-browser:1.7.0
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8080
          envFrom:
            - secretRef:
                name: docker-registry-ui-secrets-sealed-secrets
```

`modules/docker-registry-system/service.yaml`

```yaml
... 既存のservice
---
apiVersion: v1
kind: Service
metadata:
  name: docker-registry-ui-service
  namespace: docker-registry-system
spec:
  selector:
    app: docker-registry-ui-app
  type: ClusterIP
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
```

`modules/docker-registry-system/ingress.yaml`

```yaml
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: docker-registry-ui-ingress
  namespace: docker-registry-system
spec:
  tls:
    certResolver: le
  routes:
    - kind: Rule
      match: Host(`UIを公開するURLの例:docker-ui.lemon.com`)
      services:
        - name: docker-registry-ui-service
          port: 80
```

加えてBasic Authの設定を行いたい場合は、 [Sealed Secretsによるシークレット管理 + Traefik Basic Auth設定](/ja/p/1379182f14a62e9b/) を参考にしてください。


`modules/docker-registry-system/sealed-docker-registry-ui-secrets.yaml`

sealed secretsのGUIに入って、次のSecretを追加します。（参考Docs [Link](https://github.com/klausmeyer/docker-registry-browser/blob/master/docs/README.md)）

- BASIC_AUTH_USER : registryへのログインに使用するユーザー名
- BASIC_AUTH_PASSWORD : パスワード
- DOCKER_REGISTRY_URL : https://<登録したRegistryのアドレス>
- ENABLE_DELETE_IMAGES : true
- SECRET_KEY_BASE : `openssl rand -hex 64` を実行して得られた値

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: docker-registry-ui-secrets
  namespace: docker-registry-system
  annotations: {}
spec:
  encryptedData:
    BASIC_AUTH_USER: afdfads...
    BASIC_AUTH_PASSWORD: fadvbads..
    DOCKER_REGISTRY_URL: dfd..
    ENABLE_DELETE_IMAGES: afdgd...
    SECRET_KEY_BASE: dffdf...

```

使い方は直感的なので、ぜひご自身で試してみることをおすすめします！


### 3. Registry GC

Docker Registry UIで画像を削除しても（あるいはCLIで画像を削除しても）、ディスク容量がすぐに減るわけではありません。

これはGUI/CLIで画像を削除する際に、実際の物理削除が起こるのではなく、その画像のManifestのみが削除されるためです。

実際のデータ削除は、Registry GCを明示的に実行したときに行われます。

もっと詳しい話もできますが、今日はとりあえず使い方に焦点を当てているので、詳細は省略します。

（関連する内容についてもう少し気になる方は、[透明セロハン理論を通したOverlay FSの使い方とUnion Mount](https://blog.naver.com/alice_k106/221530340759) という記事（韓国語）をまず読んでから、Docker registryと画像削除などのキーワードで検索してみると良いと思います。）

* Registry GCを実行する

**!! 注意! GCの実行中はリポジトリでPull/Pushが発生してはいけません。これを先に確認してから作業してください！**

1. `kubectl get pod -n docker-registry-system` を使って、RegistryのPodを見つけます。
2. `kubectl exec -it pod/<registry-pod-name> -n docker-registry-system -- /bin/registry garbage-collect /etc/docker/registry/config.yml` を実行してGCを実行します。
3. その後、Longhorn DashboardでVolumeを選択 -> Trim Filesystemを押して、Actual Sizeを再計算してもらいます。

### 4. おわりに

これを通してDocker UI、GCの方法について見てきました。

これでPrivate Registryと管理方法ができたので、安心してサーバーを開発（？）して、お好みのサーバーを入れることができます！

次回はCloudNativePGを使ってKubernetes上でデータベースを動かす方法について見ていきます。
