---
title: "自宅でRaspberry Piクラスタを使ってデータセンターを構築する - 8. Private Docker RegistryとID/PWの設定"
slug: be354d4b5f50a3ee
date: 2023-12-10T10:08:50+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - Raspberry Pi
tags:
    - K8S
    - K3S
    - Raspberry Pi
    - Docker Registry
    - コンテナレジストリ
    - Longhorn
---

_注：筆者は韓国在住のため、本文には韓国特有の文脈が含まれることがあります。_

### 1. はじめに

せっかくインフラを整えたので、Private Docker Registryも自前で作ってみましょう。

私の場合は、このクラスタ上に個人サーバーを別途立てて運用しているので、それらのコンテナを管理するためにPrivate Registryを利用しています。

Dockerhubに課金しても構わない方や、Public Imageで十分な方は、本ガイドに従う必要はありません。

Private Repoを使うので、ID/Passwordの設定も同時に進めていきます！

### 2. インストール

今回もArgoCDでサクッとインストールしましょう！

`modules/docker-registry-system/pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: longhorn-docker-registry-pvc
  namespace: docker-registry-system
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn-ssd
  resources:
    requests:
      storage: 300Gi
```

私の場合はMLサーバーを運用していてイメージサイズが大きいので、余裕を持って300Gi程度割り当てていますが、必要に応じて調整してください。

`modules/docker-registry-system/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: registry-service
  namespace: docker-registry-system
spec:
  selector:
    app: registry
  type: LoadBalancer
  ports:
    - name: docker-port
      protocol: TCP
      port: 5000
      targetPort: 5000
  loadBalancerIP: 192.168.0.202
```

`modules/docker-registry-system/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: registry
  namespace: docker-registry-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: registry
  template:
    metadata:
      labels:
        app: registry
        name: registry
    spec:
      nodeSelector:
        node-type: worker
      containers:
      - name: registry
        image: registry:2
        ports:
        - containerPort: 5000
        env:
        - name: REGISTRY_HTTP_ADDR
          value: 0.0.0.0:5000
        - name: REGISTRY_AUTH
          value: htpasswd
        - name: REGISTRY_AUTH_HTPASSWD_REALM
          value: docker-registry
        - name: REGISTRY_AUTH_HTPASSWD_PATH
          value: /auth/registry.password
        - name: REGISTRY_STORAGE_DELETE_ENABLED
          value: "true"
        volumeMounts:
        - name: lv-storage
          mountPath: /var/lib/registry
          subPath: registry
        - name: docker-registry-account-htpasswd
          mountPath: /auth
          readOnly: true
      volumes:
        - name: lv-storage
          persistentVolumeClaim:
            claimName: longhorn-docker-registry-pvc
        - name: docker-registry-account-htpasswd
          secret:
            secretName: docker-registry-account-htpasswd
```

docker-registry-account-htpasswdというSecretを通じて、Registryで使用するID/Passwordを指定します。詳細は後述します。

`modules/docker-registry-system/sealed-docker-registry-account-htpasswd.yaml`

Secretsの設定は少々複雑です。Step By Stepで進めていきましょう！

1. Control Nodeで`htpasswd -Bbn <id> <password>`を実行し、Docker Registryで使うid/password文字列を取得します。
2. 7-1で構築したkubeseal-webguiを使って、そのSecretを暗号化します。
![例](image.png)
1. 暗号化されたSecretをsealed-docker-registry-account-htpasswd.yamlに貼り付けます。

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: docker-registry-account-htpasswd
  namespace: docker-registry-system
  annotations: {}
spec:
  encryptedData: 
    registry.password: dfadf...
```

`modules/docker-registry-system/ingress.yaml`
```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: docker-registry-ingress
  namespace: docker-registry-system
spec:
  tls:
    certResolver: le
  routes:
    - kind: Rule
      match: Host(`<使用するsubdomain>`)
      services:
        - name: registry-service
          port: 5000
```

### 3. 動作確認

正常に動作することが確認できたらデプロイします！

デプロイが完了したら、id/passwordで正常にログインできるか確認します。

dockerがインストールされている環境のターミナルで、以下を実行します。

```shell
docker login <my-registry.lemon.com>
```

その後id/passwordを入力し、正常にログインできるか確認します。

### 4. Private Registryからイメージを取得する方法

ID/Passwordが必要なPrivate Registryからイメージをpullする方法を知っておく必要がありますね。そうしないと、せっかくpushしたイメージが使えませんからね。

私が使っている方法を一旦共有します。もっとスマートなやり方がありそうなので、ご存知の方はぜひ教えてください！

例えばauth-server-systemというnamespaceで使う場合：

1. `kubectl create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>`を使って、default Namespaceにregcredというsecretを作成します。
2. `kubectl get secret regcred --output=yaml > secret.yaml`でsecret.yamlにsecret情報を保存します。このSecretはどこかにバックアップしておきます。
3. `kubectl delete secret regcred`を実行して、default namespaceに作成したsecretを削除します。
4. その後、2で保存したSecretのNamespaceを、使用したいnamespaceに変更します（例：auth-server-system）。

```yaml
apiVersion: v1
data:
  .dockerconfigjson: ddfadf...
kind: Secret
metadata:
  name: regcred
  namespace: auth-server-system # 毎回ここを変更
type: kubernetes.io/dockerconfigjson
```

5. その後`cat secret.yaml | kubeseal --controller-namespace=sealed-secrets-system --controller-name=sealed-secrets -oyaml > sealed-regcred.yaml`コマンドでSealed Secretを生成します。
6. Sealed regcredをデプロイ時に一緒にデプロイし、Deploymentに以下のようにImagePullSecretを追加します。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-server
  namespace: auth-server-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: auth-server
  template:
    metadata:
      labels:
        app: auth-server
        name: auth-server
    spec:
      nodeSelector:
        node-type: worker
      containers:
        - name: auth-server
          image: registry.lemon.com/auth_server_repository:2023.12.10.1
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 80
      imagePullSecrets:
        - name: regcred # これを追加
```

### 5. おわりに

おめでとうございます！これでPrivate Docker Registryの構築が完了し、

Id/Passwordで自分のイメージを安全に守れるようになりました！

次回はWeb UIを通じてイメージを管理する方法と、Docker GCの方法についてご紹介します。
