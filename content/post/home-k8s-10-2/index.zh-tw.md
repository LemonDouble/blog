---
title: "在家用樹莓派叢集搭建資料中心 - 10-2. 使用 Forward Auth 與 OAuth2 實作 SSO"
slug: a63eb8199e2089b6
date: 2024-01-07T16:05:04+09:00
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
    - Forward Auth
    - OAuth2
    - SSO
    - 認證
---

_註：筆者居住於韓國，部分內容包含韓國特有的背景。_

### 1. Forward Auth?

當我們需要認證邏輯時，可以在哪裡實作認證邏輯呢？

最先想到的方法，可能是直接在應用程式中實作。

但如果應用程式變多了呢？或者部分應用程式並不是我自己開發的，而是別人開發的，我沒辦法隨意修改其中的認證邏輯，那該怎麼辦？

我們正在使用 Kubernetes，所有請求都會經過 Traefik 一次。

那麼，讓 Traefik 在`「轉發請求之前」`處理認證邏輯如何？

![Traefik forward Auth 圖](image.png)

**很簡單！**

1. 所有請求都會經過 Traefik。
2. 如果配置了 Forward Auth，則向外部認證服務發送請求。
3. 外部認證服務執行如下動作：
   - 如果是有效請求，則回傳 2XX OK 代碼。這種情況下請求會正常繼續。
   - 如果是無效請求，則原樣回傳認證伺服器的回應。（回傳錯誤）

那麼我們怎麼用這個呢？

### 2. OAuth2 與 Cookie、Forward Auth

#### 2-1. 最簡單的 Forward Auth 伺服器

讓我們從頭慢慢思考。

首先，思考一下最簡單的 Forward Auth 伺服器。

要讓某些請求通過，某些請求被拒絕，應該怎麼做？

沒錯！正如我們在 Basic Auth 時看過的那樣，每個請求都附帶認證資訊，如果該認證資訊有效就讓它通過即可。

畫成圖的話，最簡單的範例如下。

![最簡單的 Forward Auth](image-1.png)

#### 2-2. 認證 Token 放在哪裡？

很好！

我們已經想出了最簡單的 Forward Auth Server。那麼接下來的問題是：「那 Token 什麼時候、在哪裡取得？」

簡單的方式可以是建立一個登入頁面，登入成功後核發 Token。

畫成圖看看。

![新增登入頁面](image-2.png)

咦？但是如果在`auth.lemondouble.com`登入，怎麼共用 Token 呢？應該把 Token 放在哪裡？

而且我還需要在 a.lemondouble.com、b.lemondouble.com 等其他網站也使用該 Token 來保持登入。

（其實這種情況還挺常見的）如果在 a.lemondouble.com 已經完成 SSO 登入，但 b.lemondouble.com 又要我重新登入的話，（即使使用者資訊共用）也會非常麻煩！

為了解決這個問題，我們打算把 Token 放到 Cookie 裡。

例如，在`.lemondouble.com`上設定 Cookie，那麼這個 Cookie 在存取所有子網域，也就是`a.lemondouble.com`、`auth.lemondouble.com`、`'b.lemondouble.com'`時會自動一起傳送。我們就利用這一點。

```javascript
res.cookie('forward-auth-token', token, {
    domain: '.lemondouble.com',        //subdomain까지 공유하려면 앞에 "."을 붙여줘야함!
});
```

這樣一來，許多事情都會變得簡單！整理一下。

1. Forward Auth 在有 Token 時允許，沒有 Token 時拒絕。
2. 建立登入頁面，登入頁面在登入成功時對`.lemondouble.com`設定 Cookie。

整理到這裡，可以畫出如下示意圖。

![登入之後到儲存 Cookie 的圖](image-3.png)

#### 2-3. 使用 JWT Token 及過期、有效性驗證

那麼！Token 裡面應該放什麼值好呢？

這種情況下，JWT 看起來比較合適。無狀態、且非常難以偽造的 Token 正好適合這種使用情境。

裡面的內容嘛……使用者 ID 或電子郵件大概比較合適。

但是 Token 一旦核發就能用一輩子嗎……？那麼 Token 一旦被竊，伺服器就一輩子被攻陷了……？所以需要設定 Token 有效期。要在安全性與便利性之間權衡，為了方便，我們假設使用 3600 秒（1 小時）。

那麼

1. JWT 的 Exp（Expired Time）應該由登入伺服器設為目前時間 + 3600 秒核發。為了方便，把 Cookie 保留時間也統一為 3600 秒比較好。
2. 之前 Forward Auth 是只要有 Token 就一律放行，但現在需要新增 JWT 驗證邏輯，並且即便值有效，過期 Token 也要拒絕。

再畫一次圖。

![新增 JWT Token 與過期機制](image-4.png)

#### 2-4. 重新導向處理

但是如果 Token 無效，或者沒有 Token，僅僅顯示錯誤頁面真的沒問題嗎？

當然……我自己用的話也沒什麼問題……但每次都會非常麻煩。如果認證失敗時自動重新導向到登入頁面會更方便。

馬上加上去吧。

![新增重新導向](image-5.png)

但是通常 SSO 成功後會回到原本的網站，對吧？這個也加上看看。

我們可以透過查詢參數或 Header 等方式記住「原本是從哪裡來的」，登入成功後登入伺服器再依據這個記錄做重新導向處理就行了。

![回到原網站](image-6.png)

啊……OAuth 還沒開始講就已經很複雜了。先再來一遍吧……

如果整合認證是 ID/Password 登入，那麼到這裡就完成了。這裡可能有點容易混淆，整理一下：登入伺服器和 Traefik 的 Forward Auth 伺服器可以是同一個。

之所以畫圖重新寫一次，是因為 Forward Auth 認證伺服器和登入伺服器沒有必要分開！

![失敗時流程重新整理的圖](image-7.png)

#### 2-5. OAuth 開始！

先確定規格再繼續吧。

我們要做的規格如下。

1. 如果存取某個服務，並且需要 OAuth SSO，則直接跳轉到對應 Social Provider 的登入頁面。（也就是說，我們不做登入頁面！）
2. 登入成功後，重新導向到原本要存取的服務！

至於 Client ID 註冊等基本內容就省略了。一一說明的話能寫成一本書……如果對 OAuth 2.0 協定不太了解的話，推薦生活編程的 [Web2 - OAuth 2.0](https://opentutorials.org/course/3405) 課程。[^1]（免費的！）

#### 2-6. OAuth 完整流程

我……真的努力想把這個講簡單一點……但老實說也想不到好辦法……

直接給各位看完整流程。

![OAuth SSO 完整流程](image-8.png)

核心想法是利用 State Code！

- 向 OAuth 認證伺服器發送請求時，產生一個任意字串一起發送（?state=任意字串），之後使用者登入成功返回時，會把該字串一起送回來——這是 OAuth 2.0 標準中的規範。（不是必填參數，所以你可能在做 OAuth 整合時沒注意到！）
- 利用這個機制，把使用者「從哪裡來」記錄到資料庫裡，登入成功後再重新導向到該 URL。

完整流程請參考上圖。那麼我們的認證伺服器需要實作什麼呢？

只需實作一個端點，但要實作以下邏輯。（依上面的順序處理。處理順序很重要！）

1. 如果查詢參數中有 code 和 state，檢查 State Code 是否與 DB 中儲存的值一致。如果該 State Code 存在於 DB 中，則與第三方 OAuth Server 通訊，把 Authorization Code 轉換為使用者資訊。檢查轉換後的使用者資訊是否在預先註冊的權限清單中，如果不在則回傳錯誤。如果在權限清單中，產生新的登入 JWT，並以`.yourdomain.com`存到 Cookie 裡。這時為了安全性，最好啟用 secure、httpOnly。此外 sameSite 選項設為`'Lax'`。然後將使用者重新導向到與 State Code 配對的 URI。

2. 如果 Cookie 中有 JWT Token，則在 JWT 驗證後，如果是合法的 JWT 則回傳 200。


3. 如果 Cookie 中根本沒有 JWT Token，或者是過期的 Cookie，則刪除過期 Cookie，解析 `X-Forwarded-Host` 標頭和 `X-Forwarded-Uri` 以擷取目前 URI。然後把新的 State Code 和該 URI 配對存入 DB，把目前 API 設為 Redirect URL，並連同 State Code 一起發送。

- 提示：使用者請求的 URI：`https://" + GetHeader("X-Forwarded-Host") + GetHeader("X-Forwarded-Uri")`

#### 2-7. 額外設定

最後……在 Traefik 設定中加入以下兩行。

```
- "--entryPoints.web.forwardedHeaders.insecure"
- "--entryPoints.websecure.forwardedHeaders.insecure"
```

沒有這兩行，X-forwarded-for 標頭就不會傳過來！

如果您參考過我部落格的文章（[4. Traefik HTTPS 認證及 ArgoCD 設定](/zh-tw/p/%EC%A7%91%EC%97%90%EC%84%9C-%EB%9D%BC%EC%A6%88%EB%B2%A0%EB%A6%AC-%ED%8C%8C%EC%9D%B4-%ED%81%B4%EB%9F%AC%EC%8A%A4%ED%84%B0%EB%A1%9C-%EB%8D%B0%EC%9D%B4%ED%84%B0%EC%84%BC%ED%84%B0-%EC%B0%A8%EB%A6%AC%EA%B8%B0-4.-traefik-https-%EC%9D%B8%EC%A6%9D-%EB%B0%8F-argocd-%EC%84%A4%EC%A0%95/)），可以這樣設定。

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
      - "--entryPoints.web.forwardedHeaders.insecure"
      - "--entryPoints.websecure.forwardedHeaders.insecure"
```

之後執行 kubectl apply，完成 Forward Auth 設定！

### 結語

其實內容實在太龐大……我已經盡量寫得簡單了，但估計也沒能好好傳達。以後大概要再修改一次這篇文章。

不過以防萬一，如果有要參考著搭建的人，遇到不理解的地方，請隨時在下方留言或透過 contact@lemondouble.com 聯絡我！我會一起幫忙排查問題。

啊，對了，如果您會 Golang，也可以參考 [traefik-forward-auth](https://github.com/thomseddon/traefik-forward-auth)。

[^1]: 生活編程（생활코딩）是韓國的一個熱門線上學習平台，主要以韓語提供免費的程式設計課程。
