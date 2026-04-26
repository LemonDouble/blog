---
title: "在家用樹莓派叢集架設資料中心 - 4. Traefik HTTPS 認證及 ArgoCD 設定"
slug: f10efbc0bcc4c537
date: 2023-12-06T05:21:00+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - 樹莓派
tags:
    - K8S
    - K3S
    - 樹莓派
    - Traefik
    - Let's Encrypt
    - HTTPS
    - ArgoCD
    - GitOps
---

_註：筆者居住於韓國，部分內容包含韓國特有的背景。_

如果透過 Git 來管理整個系統會怎麼樣呢？
將所需的容器種類、數量等全部宣告在 Git 中，叢集完全按照 Git 中宣告的內容運作，這樣不是非常方便嗎？

不必特地查看叢集的各個角落，只需查看上傳到 Git 的設定值即可。

而且，如果在部署過程中發生問題，不就可以立即 Revert 嗎？此外，多人管理同一叢集時產生的問題也可以透過我們經常使用的 PR 來管理。

最後，如果叢集突然出現問題導致所有節點當機，由於宣告的狀態值在 Git 中，即使透過其他叢集也能快速進行復原。

這種方法被稱為 GitOps，而 ArgoCD 是幫助將 Git 中宣告的狀態與叢集狀態保持一致的工具。

接下來需要部署的系統真的非常多，所以讓我們先安裝幫助實現 GitOps 的 ArgoCD。

同時為了讓 ArgoCD 可以從外部存取，讓我們一起進行 Traefik 的 LetsEncrypt 設定等工作。

### 套用 Traefik Letsencrypt Certification

- 參考文章：[HTTPS on Kubernetes Using Traefik Proxy](https://traefik.io/blog/https-on-kubernetes-using-traefik-proxy/)

在 Control Node 上參考以下內容建立一個 traefik-update.yaml。

```yaml
# https://traefik.io/blog/https-on-kubernetes-using-traefik-proxy/
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    ports:
      web:
        redirectTo:
          port: websecure
          priority: 10
    additionalArguments:
      - "--log.level=INFO"
      - "--certificatesresolvers.le.acme.email=yourMail@gmail.com"
      - "--certificatesresolvers.le.acme.storage=/data/acme.json"
      - "--certificatesresolvers.le.acme.tlschallenge=true"
      - "--certificatesresolvers.le.acme.caServer=https://acme-v02.api.letsencrypt.org/directory"
```

接著使用 `kubectl apply -f traefik-update.yaml` 套用到叢集。

之後，在 IngressRoute 中可以使用 le（letsEncrypt）作為 CertificationResolver。（如果不明白這是什麼意思，繼續操作下去就會理解。）

### ArgoCD 安裝

讓我們安裝 ArgoCD，並串接 Github SSO。

1. 進行 DNS 設定。將從外部存取 ArgoCD 時使用的子網域與目前的 IP 位址關聯。
    - 造訪「查看我的 IP」服務確認自己的 IP。
    - 進入購買 DNS 的網站，在 A 紀錄的 IP4 Address 中填入剛才確認的自己的 IP。

2. 建立一個 `Github Organization`，並建立合適的 Role。我的情況是在個人 Organization 中新增了名為 Admin 的 Role。

3. 進入 Organization -> Settings -> Developer Settings -> OAuth Apps，新增一個新的 OAuth 應用程式。

![ArgoCD OAuth 設定，Authorization callback URL 很重要](image.png)

此時，將 Authorization callback URL 設定為 `https://<剛才設定的網域>/api/dex/callback`。
其餘可以自由變更。

之後發放並記住 `Client ID, Secret`。

4. 建立 values.yaml 檔案

參考以下檔案撰寫 `values.yaml` 檔案。

```yaml
configs:
  params:
    # 由於 Traefik 處理 Https 連線，因此以 Insecure 模式啟動
    server.insecure: true
  rbac:
    # Built-in Policy : https://github.com/argoproj/argo-cd/blob/master/assets/builtin-policy.csv
    policy.default: role:readonly
    policy.csv: |
      g, <Github Organization 名稱>:<建立的 Role 名稱>, role:admin

server:
  config:
    url: https://<設定的網域名稱>

    dex.config: |
      connectors:
      - type: github
        id: github
        name: github
        config:
          clientID: <剛才發放的 Client ID>
          clientSecret: <剛才發放的 Client Secret>
          orgs:
          - name: <Github Organization 名稱>
```

例如：

```yaml
configs:
  params:
    # 由於 Traefik 處理 Https 連線，因此以 Insecure 模式啟動
    server.insecure: true
  rbac:
    # Built-in Policy : https://github.com/argoproj/argo-cd/blob/master/assets/builtin-policy.csv
    policy.default: role:readonly
    policy.csv: |
      g, LemonLemon:Admin, role:admin

server:
  config:
    url: https://argo.lemon.com

    dex.config: |
      connectors:
      - type: github
        id: github
        name: github
        config:
          clientID: vadf45a6sdfad
          clientSecret: adsfvasd45f6ads4f56asd7
          orgs:
          - name: LemonLemon
```

5. 建立 ingress.yaml 檔案

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: argocd-ingress
  namespace: argocd
spec:
  entryPoints:
    - websecure
  routes:
    - kind: Rule
      match: Host(`<設定的網域名稱>`)
      priority: 10
      services:
        - name: argo-cd-argocd-server
          port: 80
    - kind: Rule
      match: Host(`<設定的網域名稱>`) && Headers(`Content-Type`, `application/grpc`)
      priority: 11
      services:
        - name: argo-cd-argocd-server
          port: 80
          scheme: h2c
  tls:
    certResolver: le
```

以下是範例。

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: argocd-ingress
  namespace: argocd
spec:
  entryPoints:
    - websecure
  routes:
    - kind: Rule
      match: Host(`argo.lemon.com`)
      priority: 10
      services:
        - name: argo-cd-argocd-server
          port: 80
    - kind: Rule
      match: Host(`argo.lemon.com`) && Headers(`Content-Type`, `application/grpc`)
      priority: 11
      services:
        - name: argo-cd-argocd-server
          port: 80
          scheme: h2c
  tls:
    certResolver: le
```

6. 撰寫完 `values.yaml` 與 `ingress.yaml` 後，輸入以下指令安裝 argocd。

```sh
kubectl create namespace argocd
helm repo add argo https://argoproj.github.io/argo-helm
helm install -n argocd argo-cd argo/argo-cd -f values.yaml

kubectl apply -f ingress.yaml
```

稍等片刻後，存取設定的網域（我的情況是 `https://argo.lemon.com`），ArgoCD 會正常出現。

之後點擊 `Log in Via Github` 按鈕登入。

如果登入成功，則表示安裝成功！

※ 提示：如果出現 401 錯誤，請嘗試 kill 一次 argocd 命名空間的 argocd-server pod。

### ArgoCD Git Private Repo 連線

如果叢集使用 Public Repo，可以省略此設定，但為了防止萬一發生事故，讓我們在 Git 中也使用 private Repo，並在 ArgoCD 中連接使用。

在 Github 中新增一個新的（空的）儲存庫後，將該儲存庫與 ArgoCD 串接。

1. 登入 ArgoCD 後，點擊 Settings -> Repository -> Connect Repo。
2. connection method 選擇 ssh，Name 任意，Project 為 default，Repository URL 填入 git clone 時的 `ssh 位址`，SSH private key data 複製貼上 Control01 節點的 `~/.ssh/id_rsa`。
3. 連線 Git。

![Github 串接範例](image-1.png)

### 結語

今天我們完成了 Traefik Ingress 設定、ArgoCD 安裝及 Git 串接。

下一篇文章將整理至今散落各處(?)的 yaml 檔案，使用 App of Apps 整理儲存庫結構，並建構一個即使發生故障也能透過一次點擊復原的系統。

一如既往，如有錯誤請告知！
