---
title: "在家用樹莓派叢集打造資料中心 - 7. 透過 Sealed Secrets 進行密鑰管理 + Traefik Basic Auth 設定"
slug: 1379182f14a62e9b
date: 2023-12-10T00:41:05+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - 樹莓派
tags:
    - K8S
    - K3S
    - 樹莓派
    - Sealed Secrets
    - 密鑰管理
    - kubeseal
    - Traefik
---

_註：筆者居住於韓國，部分內容包含韓國特有的背景。_

### 1. Sealed Secrets?

我們現在正在做 GitOps。也就是說，我們把所有與叢集相關的設定都上傳到 Git。

可是，Kubernetes Secret 沒有加密功能。任何人用 base64 解碼後都能看到明文內容。

那這樣就不能上傳到 Git 了對吧?

為了解決這種問題，Sealed Secrets 應運而生。

1. 叢集擁有加密/解密金鑰。
2. 使用者在把檔案上傳到 Git 之前，先用叢集所持有的金鑰進行加密。
3. 加密後的 Secret 被上傳到 Git。任何人都能看到，但無法解密，因此是安全的。
4. 之後透過 ArgoCD 等將該 Secret 重新載入到叢集時，Sealed Secret 控制器會解開加密並自動解密。

Sealed Secrets GitHub : [Link](https://github.com/bitnami-labs/sealed-secrets)

### 2. 加入 Sealed Secrets

透過 ArgoCD 一氣呵成地完成部署吧!

`apps/enabled/sealed-secrets-system.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sealed-secrets-system
  namespace: argocd
spec:
  destination:
    namespace: sealed-secrets-system
    server: 'https://kubernetes.default.svc'
  source:
    path: modules/sealed-secrets-system
    repoURL: 'git@github.com:<YourOrganizationName>/<YourRepositoryName>.git'
    targetRevision: HEAD
  project: default
```

`modules/sealed-secrets-system/sealed-secrets.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sealed-secrets
  namespace: argocd
spec:
  destination:
    namespace: sealed-secrets-system
    server: 'https://kubernetes.default.svc'
  source:
    repoURL: 'https://bitnami-labs.github.io/sealed-secrets'
    targetRevision: 2.13.3
    chart: sealed-secrets
    helm:
      releaseName: sealed-secrets
  project: default
```

接著進行部署。

接下來，使用以下指令安裝 kubeseal 命令列工具。

```shell
KUBESEAL_VERSION='0.23.0' #截至 2023.12 的最新版本
wget "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION:?}/kubeseal-${KUBESEAL_VERSION:?}-linux-arm64.tar.gz"
tar -xvzf kubeseal-${KUBESEAL_VERSION:?}-linux-arm64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```


`secret.yaml`
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
  namespace: mynamespace
data:
  users: bGVtb246JGFwcjEkL0ZYczZGam0kMkpJZWNlQy45UWc5QjV5NUw2TzVoMAoK
```

透過以下指令，可以建立並註冊 Sealed Secrets。
```sh
cat secret.yaml | kubeseal --controller-namespace=kube-sealed-secrets-system --controller-name=sealed-secrets -oyaml > sealed-secrets.yaml
```

### 3. 利用 Sealed Secrets 註冊 Basic Auth

Basic Auth 簡單粗暴地說，就是 ID/密碼登入。

![就是這種東西 (Id, password 登入)](image.png)

讓我們設定 Ingress 讓 Longhorn Dashboard 也能從外部網際網路存取，並加上 Id/密碼登入吧! (在 Control01 上執行)

1. 執行 `apt install apache2-utils`，使 htpasswd 指令可用。
2. 透過 `htpasswd -nb <id> <password> | openssl base64`，取得包含 id、password 的 Secret String。
3. 用任意文字編輯器建立 secret.yaml，參考以下內容加入 Secret。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: longhorn-system-basic-auth
  namespace: longhorn-system
data:
  users: <在 2 中取得的 String>
```

4.輸入 `cat secret.yaml | kubeseal --controller-namespace=sealed-secrets-system --controller-name=sealed-secrets -oyaml > sealed-secrets.yaml` 取得 sealed-secrets.yaml。
5. 參考以下內容註冊 ingress。

`modules/longhorn-system/ingress.yaml`

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: longhorn-dashboard
  namespace: longhorn-system
spec:
  tls:
    certResolver: le
  routes:
    - kind: Rule
      match: Host(`<想要的 subdomain，例如:longhorn.lemon.com>`)
      middlewares:
        - name: basic-auth
          namespace: longhorn-system
      services:
        - name: longhorn-frontend
          port: 80
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: basic-auth
  namespace: longhorn-system
spec:
  basicAuth:
    secret: longhorn-system-basic-auth
```

在 `modules/longhorn-system/sealed-basic-auth-secret.yaml` 中，把剛產生的 sealed-secrets.yaml 複製貼上進去。

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: longhorn-system-basic-auth
  namespace: longhorn-system
spec:
  encryptedData:
    users: adfasdflavasdlfj...
  template:
    metadata:
      creationTimestamp: null
      name: longhorn-system-basic-auth
      namespace: longhorn-system
```

6. 接著進行部署。
7. 之後存取時，會正常彈出 id/密碼確認視窗，不登入就看不到該頁面!

### 3. 備份與還原

現在我們也能把 Secret 這類敏感資訊存放到 Git 上了!

不過雖然一切都好，但如果叢集掛了導致解密金鑰遺失，那我們自己也解不開秘密了對吧?

因此，加密/解密金鑰**一定要**備份!

- 備份方法

1. 輸入 `kubectl get secret -n sealed-secrets-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml >master.key`，把 master.key 一定要備份到某個地方。

- 還原方法

1. 把剛才備份的 master.key 複製到叢集上，輸入 `kubectl apply -f master.key` 註冊 master key。
2. 殺掉 sealed-secrets-system 中的一個 Pod，讓它重新讀取新的 Key。(在 ArgoCD 中可以方便地做到)

### 4. 結語

我們現在透過 Sealed Secrets 可以把所有資料都儲存到 Git 上，

並且透過 Traefik Basic Auth 可以把內部服務對外公開(具備最低限度的安全性)了!

那麼下次，我們將了解如何架設 Private Docker Registry!
