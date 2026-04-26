---
title: "在家用樹莓派叢集打造資料中心 - 10-1. 使用 Basic Auth 保護內部服務"
slug: 771435ba12d3f38c
date: 2024-01-07T12:41:24+09:00
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
    - Basic Auth
    - Sealed Secrets
    - 認證
---

_註：筆者居住於韓國，部分內容包含韓國特有的背景。_

**以下內容與 Sealed Secrets [第 3 節](https://lemondouble.github.io/zh-tw/p/%EC%A7%91%EC%97%90%EC%84%9C-%EB%9D%BC%EC%A6%88%EB%B2%A0%EB%A6%AC-%ED%8C%8C%EC%9D%B4-%ED%81%B4%EB%9F%AC%EC%8A%A4%ED%84%B0%EB%A1%9C-%EB%8D%B0%EC%9D%B4%ED%84%B0%EC%84%BC%ED%84%B0-%EC%B0%A8%EB%A6%AC%EA%B8%B0-7.-sealed-secrets%EB%A5%BC-%ED%86%B5%ED%95%9C-%EB%B9%84%EB%B0%80-%EA%B4%80%EB%A6%AC--traefik-basic-auth-%EC%84%A4%EC%A0%95/#3-sealed-secrets-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC-basic-auth-%EB%93%B1%EB%A1%9D%ED%95%B4%EB%B3%B4%EA%B8%B0) 有相當多重疊的部分。如果您已經完成那一節，並且對 Basic Auth 較為熟悉，可以直接跳過。**

### 1. 前言

使用 Kubernetes 時，會陸續部署各種各樣的服務。

當然，僅在內網中提供服務在安全方面是最理想的，但叢集管理並不一定總能從內網進行。例如外出時突然出現問題之類的情況也是有可能發生的。

這種情況下大致有兩種解決方案。

1. 使用 VPN，從其他電腦連入內網，使用相關服務
2. 將服務暴露在網際網路上，但加入認證機制

我個人選擇了第 2 種，並自己建置了一個 SSO 伺服器使用。

不過，由於 SSO 系統的建置過程相當龐大，這裡先從最簡單的認證機制開始。

### 2. Basic Auth 的流程

![Basic Auth，來源：MDN](image.png)

Basic Auth 非常簡單！

1. 用戶端向伺服器請求頁面。
2. 伺服器收到請求，如果沒有認證資訊，則回應 401 並要求認證方式（Basic Auth）。
3. 用戶端收到回應後，進行 Basic Auth。
    - 此時，HTTP Authorization Header 中會帶上 `Basic ${認證 token}` 的值。
    - 此處的認證 token 是 Base64Encode("userId:password")。
    - 例如 id 為 lemondouble，密碼為 1q2w3e4r! 時，使用對 `"lemondouble:1q2w3e4r!"` 進行 base64 編碼後的值 `bGVtb25kb3VibGU6MXEydzNlNHIh`。
4. 伺服器驗證該 token 後，回傳正常回應。

### 3. Basic Auth 的優缺點

優點：

1. 內建！ 大多數瀏覽器無需額外設定即可支援。
2. 實作簡單。 大多數 Web 框架都預設支援！

缺點：

1. 安全性較弱：這是最大的缺點。
   - 密碼以 base64 編碼（=明文）的形式在每次請求中傳輸。
     - 若使用 HTTPS 協定還好，但若使用 HTTP 協定，密碼會在每次請求中以明文形式傳送！
   - 不防禦暴力破解等攻擊。
     - 這雖然也算是 Id/Password 本身的問題，但在 Traefik 的預設設定下，並沒有諸如丟棄失敗請求超過一定次數的用戶端之類的功能。
     - 如果 Id/Password 較弱，惡意攻擊者可能透過暴力破解存取該服務。

因此，如有可能，建議在以下情形下使用此功能：

1. 即使被存取也無大礙的服務，或僅希望分享給少數使用者的服務
2. 使用 HTTPS 協定
3. 使用足夠長的密碼

### 4. Basic Auth 的設定方法

- 我們將使用 Sealed Secrets 對連線資訊（Id/Password）進行加密，並儲存到 Git 中。
- 這樣做的原因如下：
  - 在突發事故發生時，連同連線資訊一起快速復原叢集（Secrets 資訊也在 Git 中，只需部署即可）
  - 即便使用 Private Repo，也可以防止因一時疏忽導致 Id/Password 外洩

Disclaimer：以下內容與 Sealed Secrets [第 3 節](https://lemondouble.github.io/zh-tw/p/%EC%A7%91%EC%97%90%EC%84%9C-%EB%9D%BC%EC%A6%88%EB%B2%A0%EB%A6%AC-%ED%8C%8C%EC%9D%B4-%ED%81%B4%EB%9F%AC%EC%8A%A4%ED%84%B0%EB%A1%9C-%EB%8D%B0%EC%9D%B4%ED%84%B0%EC%84%BC%ED%84%B0-%EC%B0%A8%EB%A6%AC%EA%B8%B0-7.-sealed-secrets%EB%A5%BC-%ED%86%B5%ED%95%9C-%EB%B9%84%EB%B0%80-%EA%B4%80%EB%A6%AC--traefik-basic-auth-%EC%84%A4%EC%A0%95/#3-sealed-secrets-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC-basic-auth-%EB%93%B1%EB%A1%9D%ED%95%B4%EB%B3%B4%EA%B8%B0) 完全相同！

讓我們將 Longhorn Dashboard 設定為可從外部存取，同時學習 Basic Auth 的設定方法。

1. 執行 `apt install apache2-utils`，以便使用 htpasswd 命令。
2. 使用 `htpasswd -nb <id> <password> | openssl base64`，取得包含 id、password 的 Secret String。
3. 用任意文字編輯器建立 secret.yaml，並參考下方內容新增 Secret。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: longhorn-system-basic-auth
  namespace: longhorn-system
data:
  users: <步驟 2 中得到的 String>
```

4. 輸入 `cat secret.yaml | kubeseal --controller-namespace=sealed-secrets-system --controller-name=sealed-secrets -oyaml > sealed-secrets.yaml`，取得 sealed-secrets.yaml。
5. 參考下方內容註冊 ingress。

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
      match: Host(`<想要的 subdomain，例如 longhorn.lemon.com>`)
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

在 `modules/longhorn-system/sealed-basic-auth-secret.yaml` 中，將剛剛產生的 sealed-secrets.yaml 複製貼上進去。

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

6. 之後進行部署。
7. 存取時會正常彈出 id/password 確認視窗，未登入則無法看到該頁面！

### 5. 結語

Basic Auth 是我早期的叢集管理方式。

如果對外暴露的服務不多，我覺得僅僅設定 Basic Auth 並使用 [Bitwarden](https://lemondouble.github.io/zh-tw/p/fly.io-%EC%86%8C%EA%B0%9C-%EB%B0%8F-fly.io%EC%97%90-%EC%98%AC%EB%A6%AC%EA%B8%B0-%EC%A2%8B%EC%9D%80-%EC%84%9C%EB%B9%84%EC%8A%A4-%EC%B6%94%EC%B2%9C-vaultwarden/) 等密碼管理器來管理就已經足夠了。

但如果之後專案越來越多呢？

為了因應這種情況，我會在下一篇文章中介紹 SSO 認證方式。

但由於認證伺服器是我自己 DIY 的，可能很難提供完整的原始碼，還請各位先包涵，我會先介紹可直接使用的 Basic Auth 🙇
