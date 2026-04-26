---
title: "搞壞三張SD卡後氣炸了寫的樹莓派USB Boot設定方法"
slug: 977ba9d70a2a7029
date: 2023-01-29T17:31:39+09:00
image: cover.png
categories:
    - 樹莓派
tags:
    - 樹莓派
    - USB Boot
    - EEPROM
    - SD卡
---

_註：筆者居住於韓國，部分內容包含韓國特有的背景。_

樹莓派哪裡都好，就是SD卡開機經常讓人非常火大。

折騰的時候SD卡反覆拔插，不小心刮傷了導致無法辨識，真的會讓人心煩意亂。

我知道SD卡和USB用的是類似的技術，但是……

經驗上，USB的穩定性通常要好一些（實體損壞方面）。

所以，把SD卡扔掉，讓我們用USB來開機吧！


`以下內容基於Ubuntu 22.04 Server環境撰寫。`

### 1. 燒錄SD卡

* 要從USB開機，得先調整bootloader，而這就需要至少用SD卡開機一次。
* 燒錄SD卡，進行SSH連線。之後的指令都在終端機裡執行。

### 2. 更新EEPROM中的Bootloader

* EEPROM：還記得在計算機結構課上學到的ROM嗎？大致上把它當作「可寫可擦的ROM」就行。
* Bootloader就放在這裡執行，大約2020年9月之後版本的bootloader開始支援USB開機。
* 順便升級一下吧。

Ref: https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#automaticupdates

1. 執行 `sudo rpi-eeprom-update` 指令，確認當前/最新版本的Bootloader。
2. 執行 `sudo rpi-eeprom-update -a` 指令，預約Bootloader更新。
3. 用 `sudo reboot` 重新開機，開機過程中bootloader更新就會執行。
4. 再次執行 `sudo rpi-eeprom-update` 指令，確認更新是否成功。


- Update (2023.05.19)
  - 上面的 rpi-eeprom-update 只支援RPI4以上。
  - 如果你使用的是RPI 3B或更早的版本，請參考 [Booting Raspberry Pi 3 B With a USB Drive](https://www.instructables.com/Booting-Raspberry-Pi-3-B-With-a-USB-Drive/)。
  - 如果使用Raspbian OS，按照該指南操作即可。
  - 如果使用Ubuntu，config.txt 需要修改 `boot/firmware/config.txt`！！！


### 3. 設定開機順序

* Ubuntu Server裡沒有raspi-config程式，所以先安裝。

1. 輸入 `sudo apt install raspi-config` 來安裝raspi-config程式。
2. 輸入 `sudo raspi-config` 指令後，進入 Advanced Option -> Boot Order。
3. 選擇 `USB Boot`，然後用 `sudo reboot` 重新開機。

### 4. 確認開機順序是否正確套用

* 輸入 `vcgencmd bootloader_config`。
* 檢查下方是否設定了 `BOOT_ORDER=0xf14`。如果沒有設定，表示設定過程中出了問題，重複1~3步。
* Ref: https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#configuration-properties
![](2023-01-29-18-07-12.png)

### 5. 在USB上燒錄OS後插上重新開機

* 拔掉SD卡，在USB上燒錄OS後重新開機。
* 哇！開機成功了。Good！
