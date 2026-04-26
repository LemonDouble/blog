---
title: "使用 Private DNS 與 ALB 建構從 VPC 外部無法存取的 API"
slug: acf607c351652785
date: 2023-01-22T17:12:43+09:00
image: cover.png
categories:
    - AWS
    - MSA
tags:
    - AWS
    - Route 53
    - ALB
    - VPC
    - Private DNS
    - 網路
---

_註：筆者居住於韓國，部分內容包含韓國特有的背景。_

建構 MSA 服務時，伺服器之間會發生非常頻繁的通訊。

舉例來說，假設有一個（高度簡化的）轉帳服務。

![](2023-01-22-17-22-23.png)

如果有錢包伺服器和使用者資訊伺服器，呼叫轉帳 API 時會發生上圖中的情況。


那麼，如果**正常使用者檢查 API**可以從外部存取呢？

![](2023-01-22-17-27-08.png)

如果惡意使用者有意存取，他們甚至可以將我們服務的正常使用者比例繪製成圖表（..）

因此，在 MSA 架構中出現了這樣的需求：`伺服器之間的請求可以使用該 API，但外部使用者不應該能夠存取`。

如何才能實現這個需求呢？


### 1. 最簡單的方法是伺服器之間共享密鑰。

![](2023-01-22-17-57-32.png)

* 透過環境變數等方式，伺服器之間共享相同的密鑰。
* 將其放入 HTTP 自訂 Header 等，伺服器在請求時一起傳送密鑰。
* 伺服器收到內部 API 請求後會檢查密鑰，如果不一致則 Reject。

但這種方法存在問題。

* 如果有人偷走了密鑰怎麼辦？或者，如果密鑰洩漏到 Git 等外部儲存庫怎麼辦？
* 當然，從外部就可以發起 API 請求了。

### 2. 那麼管理 Whitelist 怎麼樣？

![](2023-01-22-18-03-55.png)

* 如果能知道所有伺服器的 IP，可以透過 `X-Forwarded-For` 等 Header 檢查 IP，如果不是我們已知的伺服器 IP 就阻擋。

但這種方法也有些問題！

* 如果請求量太大需要擴增伺服器怎麼辦？
* 當然不知道新伺服器的 IP，所以會被阻擋。

### 3. 透過網路設定阻止外部進入吧！

* 1、2 的方法也是好方法，但如果外部存取時回應根本到不了伺服器會更安全。
* 讓我們用 Private DNS / Public DNS 來實作這個！
* 這種方法也稱為 `Split Horizon DNS` 或 `Multiview DNS`。
* 相關文件：( [Link](https://aws.amazon.com/ko/premiumsupport/knowledge-center/internal-version-website/) )
* 那麼讓我們一步步了解。

* **1. 首先了解一下 Public DNS / Private DNS。**

![](2023-01-22-19-02-46.png)

* 簡單來說，從 VPC 外部存取時 / 從 VPC 內部存取時可以回傳不同的回應位址。
* 可以理解為從 VPC 內部存取時使用內部 DNS 伺服器，從外部存取時使用外部 DNS 伺服器！

![](2023-01-22-19-07-27.png)

* 那麼，如果請求內部 DNS 中沒有的記錄會怎樣？
* 如上圖所示，先查詢 Private DNS，如果沒有則查詢 Public DNS。
* (Note：可能與實際網路流程不完全一致。但重要的是，Private DNS 具有優先權。)

* **2. 那麼現在了解一下 ALB。**

![](2023-01-22-19-15-44.png)

* ALB (Application Load Balancer) 可以評估傳入的請求，並適當地分配。
    * 可以根據 HostName 適當回應。例如..
        * 進入 `wallet.lemondouble.com` 可以連接到 `wallet server`。
        * 進入 `user.lemondouble.com` 可以連接到 `user server`。
    * 使用 Path 也可以做同樣的事情。
        * 進入 `api.lemondouble.com/wallet` 可以連接到 `wallet server`。
        * 進入 `api.lemondouble.com/user` 可以連接到 `user server`。
        * 當然，也可以做到 `如果包含特定 Path 就 Deny`。


* **3. 那麼把這兩個組合起來？**

![](2023-01-22-19-26-25.png)

* 首先，建立 Public DNS 和連接到網際網路的 Public ALB。
    * 此時，Public ALB 的規則如下。
    * 優先順序 1. `如果 Hostname 與 wallet.lemondouble.com 一致且 Path 為 /internal* 則回傳固定 404 回應`
    * 優先順序 2. `如果 Hostname 與 wallet.lemondouble.com 一致則連接到 Wallet Server 的 80 連接埠`

* 此時由於 ALB 規則按優先順序應用
    * `wallet.lemondouble.com/api/hello` 不符合 `優先順序 1`，符合 `優先順序 2`，因此連接到 wallet Server。
    * `wallet.lemondouble.com/internal/hello` 符合 `優先順序 1`，因此收到 404 回應。

![](2023-01-22-19-34-23.png)

* 接下來，與 Public DNS 一樣，建立 Private DNS 和未連接到網際網路的 Private ALB。
    * 此時，Private ALB 的規則如下。
    * 優先順序 1. `如果 Hostname 與 wallet.lemondouble.com 一致則連接到 Wallet Server 的 80 連接埠`


* 在這種情況下，由於沒有對 `/internal*` 回傳 404 的邏輯，所有請求都會被轉送到 Wallet Server。

![](2023-01-22-19-36-14.png)

* 把兩個組合起來就是這樣的形態。
* 當呼叫 `wallet.lemondouble.com/internal/hello` API 時，從 VPC 外部會收到 404 回應，但從 VPC 內部會收到正常回應。

### 4. 那麼僅僅做網路設定就足夠了嗎？

* 取決於你建構的應用對資訊洩漏有多敏感，但結合 1 和 3 的方法會更好。這樣..
    * 即使伺服器之間的密鑰洩漏，由於網路設定的阻擋，外部存取會被阻斷。
    * 即使誤操作了網路設定，由於外部不知道密鑰，外部存取也會被阻斷。
    * 錯誤隨時可能發生，所以即使發生錯誤也能由系統阻擋的方法要好得多！！


以上就是建構 Private (或 Internal) API 的方法。

建構 Private API 的方法多種多樣。除此之外還可以使用 AWS 的 API Gateway，或者（雖然不太了解）也可以使用 K8S 或 Spring Cloud Gateway 等..？

希望大家把這個當作眾多方法中的一種來看就好。如果有錯誤的地方，請留言指出！

謝謝。
