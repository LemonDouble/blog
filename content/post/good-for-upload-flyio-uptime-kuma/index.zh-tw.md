---
title: "適合部署在 Fly.io 上的服務 (Uptime Kuma)"
slug: 7da6be543113ac58
date: 2023-01-22T20:41:31+09:00
image: cover.png
math: 
categories:
    - SaaS
    - Fly.io
tags:
    - 自架站
    - Uptime Kuma
    - 監控
    - Docker
    - Fly.io
---

_註：筆者居住於韓國，部分內容包含韓國特有的背景。_

這次要介紹的專案是一個叫做 Uptime Kuma 的服務，它可以記錄伺服器的 Uptime。

它可以監控你自己的叢集/伺服器等，當發生問題時，可以透過 Telegram、Slack、Discord、E-mail 等多種方式通知你伺服器當機了。

![](2023-01-22-20-50-49.png)

長這個樣子！

進入 GitHub 網址 ( [Link](https://github.com/louislam/uptime-kuma) ) 還可以體驗 Demo 服務。

關於 Fly.io 的介紹，我在之前的文章 ([Fly.io 介紹及適合部署在 Fly.io 上的服務推薦 (VaultWarden)](/zh-tw/p/fly.io-%EC%86%8C%EA%B0%9C-%EB%B0%8F-fly.io%EC%97%90-%EC%98%AC%EB%A6%AC%EA%B8%B0-%EC%A2%8B%EC%9D%80-%EC%84%9C%EB%B9%84%EC%8A%A4-%EC%B6%94%EC%B2%9C-vaultwarden/)) 中已經講了很多，請參考那篇文章，本篇文章只專注於安裝方法。

### 1. 建立 fly.toml 檔案

* 進入合適的目錄，輸入 `flyctl launch` 來建立 fly.toml 檔案。

### 2. 建立 volume

* 為了儲存監控紀錄、ID/Password 等，需要 Volume。
* 不需要太多的卷空間，我這邊建立的是 1GB。

```bash
flyctl volume create uptime_kuma_data --region nrt --size 1
```


### 3. 修改 fly.toml 檔案

* 請參考下面的 toml 檔案來撰寫部署檔案。
* 

```toml

# 應用名稱是 flyctl launch 時設定的值。
app = "lemon-uptime-kuma"
kill_signal = "SIGINT"
kill_timeout = 5
processes = []

# 加上 :1 表示 Debian Stable Build。
# https://hub.docker.com/r/louislam/uptime-kuma
[build]
  image = "louislam/uptime-kuma:1"

[env]

# 加上，掛載剛才建立的 Volume。
[mounts]
source="uptime_kuma_data"
destination="/app/data"

[experimental]
  auto_rollback = true

[[services]]
  http_checks = []
  internal_port = 3001 # 修改，預設設定下 Kuma 內部使用 3001 連接埠。
  processes = ["app"]
  protocol = "tcp"
  script_checks = []
  [services.concurrency]
    hard_limit = 25
    soft_limit = 20
    type = "connections"

  [[services.ports]]
    force_https = true
    handlers = ["http"]
    port = 80

  [[services.ports]]
    handlers = ["tls", "http"]
    port = 443

  [[services.tcp_checks]]
    grace_period = "1s"
    interval = "15s"
    restart_limit = 0
    timeout = "2s"
```

### 4. 部署！

```
flyctl deploy 
```
輸入該指令進行部署。

### 5. 連線並設定 ID/Password

透過 `<應用名稱>.fly.dev` 連線，
如果不清楚網址，可以連到 `https://fly.io/apps`，選擇應用並取得網址。

之後設定 ID/Password。


### 6. 新增監控

![](2023-01-22-21-01-00.png)

點擊 Add New Monitor，按以下方式設定。

然後按下 Save 按鈕..

![](2023-01-22-21-01-56.png)

監控正常運作！

### Appendix 1. TMI

* 支援韓文！可以在 Settings -> Appearance 中設定語言。

### Appendix 2. 使用 Custom Domain

* 我這邊有自己的網域，所以想做 Integration。
* 進入應用頁面，取得 IPv6。（IPv4 是 Shared IPv4，所以使用了 IPv6。）

* 進入自己的 DNS 頁面，選擇 `CNAME` 類型，值填入 `<應用名稱>.fly.dev`。
* 然後進入 https://fly.io/apps 儀表板，點擊 kuma 應用 -> Certificates -> Add Certificate，然後輸入設定好的自訂網域。

![](2023-01-22-21-32-09.png)

* 稍待片刻，就可以確認透過自己建立的自訂網域連線了！
