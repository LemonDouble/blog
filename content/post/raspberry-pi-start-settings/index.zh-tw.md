---
title: "給自己看的樹莓派初始設定"
slug: 088bdb572b7dc509
date: 2023-01-18T11:13:14Z
image: cover.png
categories:
    - 樹莓派
tags:
    - 樹莓派
    - SSH
    - Ubuntu
    - netplan
    - 初始設定
---

_註：筆者居住於韓國，部分內容包含韓國特有的背景。_

在折騰樹莓派或者家用伺服器時，意外(?)地經常需要把伺服器初始化重來。

可能是因為設定錯了導致某些東西跑不起來，或者是裝了一堆連自己都不記得裝過什麼的東西。

於是每次重置樹莓派後都有一套固定要做的工作，

但問題是每次都是隔了很久才會再初始化一次，等真要做的時候早就忘得差不多了，每次設定都得在Google裡翻半天。所以乾脆整理在這裡。


`以下內容以Raspberry Pi + Ubuntu Server為基準。初始登入密碼是 ubuntu。`


### 1. 開啟SSH存取

* 拿HDMI線接上、再接鍵盤真的非常麻煩。
* 在燒錄開機磁碟時，建立一個名為 `ssh`(無副檔名)的空檔案，開機時就會啟用SSH存取。(預設是關閉的)
* 用網路線把樹莓派接到路由器上，從路由器管理頁面拿到IP後SSH連過去。hostname會是 `ubuntu`。

### 2. SSH登入設定

* 為了能用SSH金鑰登入，把自己的公鑰加到 `~/.ssh/authorized_keys` 檔案裡。
* 如果沒有SSH金鑰，可以在Google搜尋「SSH 金鑰 登入」之類的關鍵字跟著做！

### 3. 停用密碼登入

* 如果只在內網開放的樹莓派，一般沒什麼大問題……但只要有SSH Key，從安全性角度建議盡量關掉密碼登入。
* 打開 `/etc/ssh/sshd_config`，設定 `PasswordAuthentication no`。
* 再打開 `/etc/ssh/sshd_config.d/50-cloud-init.conf`，同樣設定 `PasswordAuthentication no`。
* 然後用 `sudo systemctl reload sshd` 重啟SSH伺服器。

### 4. 連接Wi-Fi(選擇性)

* 如果一直用網路線就可以跳過。我嫌網路線讓房間顯得很亂，所以會設定一下Wi-Fi。
* 打開 `/etc/netplan/50-cloud-init.yaml` 檔案。(沒有的話就新建。)
* 把下面的檔案複製貼上進去。下面的設定檔是SSID(Wi-Fi名稱) = lemon、密碼為 lemon1234 的範例。

```yaml
# This file is generated from information provided by the datasource.  Changes
# to it will not persist across an instance reboot.  To disable cloud-init's
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
    ethernets:
        eth0:
            dhcp4: true
            optional: true
    wifis:
      wlan0:
        dhcp4: true
        optional: true
        access-points:
          "lemon":
            password: "lemon1234"
    version: 2
```

### 5. 設定Hostname

* SSH視窗裡看不到自己目前在操作哪台機器會讓人不太安心……
* 輸入 `sudo hostnamectl set-hostname <主機名稱>` 把hostname也改成容易辨識的名字。


### 6. 重新開機

* `sudo reboot`
* 透過無線網路重新連線時內部IP可能會變，連不上的話就看路由器頁面更新一下內部IP。
