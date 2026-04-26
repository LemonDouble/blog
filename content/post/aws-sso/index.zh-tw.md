---
title: "用 AWS SSO 在家也能用上公司裡那塊熟悉的登入頁（免費）"
slug: 1c9eaebbdee2ac98
date: 2023-01-21T16:17:06+09:00
image: cover.png
categories:
    - AWS
tags:
    - AWS
    - IAM Identity Center
    - SSO
    - AWS CLI
    - Organizations
---

_註：筆者居住於韓國，部分內容包含韓國特有的背景。_

如果你在公司或組織裡用過 AWS，應該看過下面這種 SSO（Single Sign On）登入頁面。

![](2023-01-21-23-57-37.png)

一旦設定好，工作階段期間不必每次都輸入 id/password，使用起來相當方便。

而且在使用 aws-cli 或 aws-sdk 時，也不必把 `AWS_ACCESS_KEY`、`AWS_ACCESS_SECRET` 這類不能外洩的資訊以明文形式存到本機電腦，在安全方面也更為有利。

加上這是免費服務（[Link](https://aws.amazon.com/ko/iam/identity-center/faqs/)），**完全** 不會產生任何額外費用！

![](2023-01-22-00-05-31.png)

那我們就來設定看看吧！


## 1. 設定 IAM Identity Center

### 1. 在 AWS 中搜尋 SSO，找到 IAM Identity Center。

![](2023-01-22-00-07-58.png)

### 2. 點擊啟用！
- 此時，Identity Center 只能在一個區域中啟用，因此在你常用的區域啟用會比較好管理。

![](2023-01-22-00-12-59.png)

### 3. 先設定 AWS Organization。

![](2023-01-22-01-00-14.png)

- 你會遇到上面這樣的錯誤訊息，請先建立 AWS Organization。
- AWS Organization 同樣是*免費*服務。（[Link](https://docs.aws.amazon.com/ko_kr/organizations/latest/userguide/orgs_introduction.html#pricing)）

### 4. 建立好 Dashboard 後，點擊「前往設定」按鈕進行後續設定。

![](2023-01-22-00-16-22.png)

### 5. 先為 SSO 登入頁面取個別名吧。
 - 預設提供的登入頁面像 `aadfbasdf-d.awsapps.com/start` 這樣自動產生，難以記憶。
 - 我們把它改成 `lemonlogin.awsapps.com/start` 這種好記的別名吧。
 - 只能設定一次，請慎重決定！

![點擊「自訂 AWS 存取入口網站 URL」。](2023-01-22-00-18-34.png)

![在這裡輸入想要的別名。](2023-01-22-00-20-02.png)

### 6. 加入群組！

![](2023-01-22-01-10-53.png)

加入群組。

### 7. 加入使用者！

![](2023-01-22-00-22-44.png)

- 點擊加入使用者按鈕後……

![](2023-01-22-00-24-25.png)

- 填寫使用者資訊。密碼可以透過電子郵件接收設定郵件，也可以使用一次性密碼。
- 我們就用預設的電子郵件方式吧。

![](2023-01-22-01-12-26.png)

- 之後可以選擇 Group，把第 6 步中設定的群組加進去。

- 然後查看電子郵件，接受邀請，為該帳號設定 Password 並記錄在某處！

### 8. 加入權限！

![](2023-01-22-00-51-29.png)

- 選擇權限集。我選的是 `AdministratorAccess`，你也可以選其他的。
- 如果預先定義的權限集中沒有想要的 Role，也可以使用自訂來直接指定。

![](2023-01-22-00-53-34.png)

- 接著填寫其餘設定。
- 工作階段時長是指經過這段時間後 AWS 會自動登出的時間。
- 預設是 1 小時，頻繁登出有點煩，我延長到了 4 小時左右。
- `中繼狀態`可以設定登入成功後重新導向的 URL。例如 Billing 權限就會在登入成功後導向 Billing 頁面。
- 如果有需要，請參考文件（[Link](https://docs.aws.amazon.com/ko_kr/singlesignon/latest/userguide/howtopermrelaystate.html)）進行設定。我留空了。


### 9. 設定 SSO 登入

![](2023-01-22-01-30-57.png)

- 接著在 AWS 帳戶頁籤中選擇自己的帳戶，點擊`指派使用者或群組`。

![](2023-01-22-01-21-14.png)

- 將上面設定的群組加進來，

![](2023-01-22-01-21-43.png)

- 再把上面設定的權限加上去。

### 10. SSO 登入

![](2023-01-22-01-33-05.png)

- 現在到第 5 步中設定的 AWS 登入網址登入看看吧。前往 `別名.awsapps.com/start` 進行登入。

![](2023-01-22-01-34-14.png)

- SSO 設定成功！


### Appendix 1. 強化安全性

![](2023-01-22-01-37-55.png)

- 進入 IAM Identity Center - 設定，可以為 SSO 帳戶設定 MFA、設定工作階段長度等。
- 即使其他維持預設，也建議開啟`若該使用者沒有已註冊的 MFA 裝置 - 登入時要求註冊 MFA 裝置`這個選項。

### Appendix 2. 透過 AWS SSO 使用 AWS_CLI

![](2023-01-22-01-47-10.png)

- 完成 SSO 設定後，按上圖使用 `aws configure sso` 進行登入設定。（紅色底線的行是手動輸入的。）
- 設定完成後，使用 `aws s3 ls` 等指令確認設定是否正常。
