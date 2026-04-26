---
title: "在家用樹莓派叢集打造資料中心 - 8. 建置私有 Docker Registry 並設定 ID/密碼"
slug: be354d4b5f50a3ee
date: 2023-12-10T10:08:50+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - 樹莓派
tags:
    - K8S
    - K3S
    - 樹莓派
    - Docker Registry
    - 容器映像倉庫
    - Longhorn
---

_註：筆者居住於韓國，部分內容包含韓國特有的背景。_

### 1. 前言

既然已經把基礎建設準備好了，我們順便也來自己建一個私有 Docker Registry 吧。

我自己的情況是，後來在這個叢集上另外部署了一些個人伺服器，為了管理這些容器映像，所以使用了私有 Registry。

如果你願意付費使用 Dockerhub，或者使用公開映像就足夠，那麼本指南可以跳過。

由於我們會使用私有 Repo，因此 ID 和密碼的設定也會一併進行！

### 2. 安裝

這次也使用 ArgoCD 快速安裝吧！

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

我自己有運行 ML 伺服器，映像體積較大，所以寬鬆地分配了 300Gi。你可以依照自己的需求調整。

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

透過名為 docker-registry-account-htpasswd 的 Secret 指定 Registry 所使用的 ID 與密碼。詳細內容後續會說明。

`modules/docker-registry-system/sealed-docker-registry-account-htpasswd.yaml`

Secrets 的設定稍微複雜一點。我們 Step By Step 來吧！

1. 在 Control Node 上執行 `htpasswd -Bbn <id> <password>`，取得 Docker Registry 要使用的 id/password 字串。
2. 使用在 7-1 中建置的 kubeseal-webgui 將該 Secret 加密。
![範例](image.png)
1. 將加密後的 Secret 貼到 sealed-docker-registry-account-htpasswd.yaml 中。

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
      match: Host(`<要使用的 subdomain>`)
      services:
        - name: registry-service
          port: 5000
```

### 3. 確認是否正常運作

確認正常運作後再進行部署！

部署完成後，確認是否能用 id/password 正常登入。

在已安裝 docker 的機器上，於終端機執行以下指令。

```shell
docker login <my-registry.lemon.com>
```

接著輸入 id/password，確認是否能正常登入。

### 4. 從私有 Registry 拉取映像的方法

接下來需要了解如何從需要 Id/Password 的私有 Registry 拉取映像，否則上傳的映像也無法使用對吧。

我先分享我自己使用的方法。也許有更乾淨的做法，如果你知道的話歡迎告訴我！

假設要在名為 auth-server-system 的 namespace 中使用：

1. 使用 `kubectl create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>` 在 default Namespace 中建立一個名為 regcred 的 secret。
2. 使用 `kubectl get secret regcred --output=yaml > secret.yaml` 將 secret 資訊存到 secret.yaml。該 Secret 請另外備份。
3. 輸入 `kubectl delete secret regcred` 刪除剛才在 default namespace 建立的 secret。
4. 之後將第 2 步 Secret 的 Namespace 改成你要使用的 namespace（例如：auth-server-system）。

```yaml
apiVersion: v1
data:
  .dockerconfigjson: ddfadf...
kind: Secret
metadata:
  name: regcred
  namespace: auth-server-system # 每次修改這裡
type: kubernetes.io/dockerconfigjson
```

5. 接著執行 `cat secret.yaml | kubeseal --controller-namespace=sealed-secrets-system --controller-name=sealed-secrets -oyaml > sealed-regcred.yaml` 指令產生 Sealed Secret。
6. 部署時一起部署 Sealed regcred，並在 Deployment 中如下加入 ImagePullSecret。

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
        - name: regcred # 加入這個
```

### 5. 結語

恭喜！現在你已經完成私有 Docker Registry 的建置，

可以透過 Id/Password 安全地保護自己的映像了！

下一篇將介紹如何透過 Web UI 管理映像，以及 Docker GC 的方法。
