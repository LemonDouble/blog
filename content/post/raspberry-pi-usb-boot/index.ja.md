---
title: "SDカードを3枚潰してブチギレた末に書く、Raspberry Pi USB Bootの設定方法"
slug: 977ba9d70a2a7029
date: 2023-01-29T17:31:39+09:00
image: cover.png
categories:
    - Raspberry Pi
tags:
    - Raspberry Pi
    - USB Boot
    - EEPROM
    - SDカード
---

_注：筆者は韓国在住のため、本文には韓国特有の文脈が含まれることがあります。_

Raspberry Piはすべてが素晴らしいのですが、SDカードからのブートが本当にイライラさせられることが多いです。

試行錯誤しているうちにSDカードを抜き差しして、どこかで擦って認識しなくなったりすると、本当に気が滅入ります。

SDカードもUSBも似たような技術を使っているのは知っていますが…

経験上、USBの方が安定性は多少マシなことが多かったです（物理的な破損の面で）。

というわけで、SDカードを投げ捨ててUSBから起動させましょう！


`以下はUbuntu 22.04 Server環境で作成されています。`

### 1. SDカードを焼く

* USBから起動させるにはブートローダーを弄る必要があり、そのためにはまず一度SDカードから起動しなければなりません。
* SDカードを焼いて、SSH接続しましょう。以降のコマンドはすべてターミナルで実行します。

### 2. EEPROMのBootloaderを更新する

* EEPROM：コンピュータアーキテクチャの授業で習ったROMですが、ざっくり「書き込みも消去もできるROM」だと思ってください。
* ここにBootloaderを載せて使うのですが、おおよそ2020年9月以降のバージョンのブートローダーからUSB起動をサポートしています。
* ついでにアップデートしておきましょう。

Ref: https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#automaticupdates

1. `sudo rpi-eeprom-update` コマンドを実行して、現在/最新バージョンのBootloaderを確認します。
2. `sudo rpi-eeprom-update -a` コマンドを実行して、Bootloaderのアップデートを予約します。
3. `sudo reboot` で再起動すると、起動中にブートローダーのアップデートが進行します。
4. `sudo rpi-eeprom-update` コマンドを実行して、アップデートが正常に行われたか確認します。


- Update (2023.05.19)
  - 上記の rpi-eeprom-update はRPI4以降のみ対応です。
  - RPI 3B以下を使用中の場合は、[Booting Raspberry Pi 3 B With a USB Drive](https://www.instructables.com/Booting-Raspberry-Pi-3-B-With-a-USB-Drive/) を参照してください。
  - Raspbian OSを使う場合は、そのガイドをそのまま辿ればOKです。
  - Ubuntuを使用中なら、config.txtは `boot/firmware/config.txt` を修正する必要があります！！！


### 3. 起動順序の設定

* Ubuntu Serverにはraspi-configプログラムが入っていないので、まずインストールします。

1. `sudo apt install raspi-config` を入力してraspi-configプログラムをインストールします。
2. `sudo raspi-config` コマンドを入力後、Advanced Option -> Boot Order に入ります。
3. `USB Boot` を選択し、`sudo reboot` で再起動します。

### 4. 起動順序がきちんと適用されたか確認

* `vcgencmd bootloader_config` を入力します。
* 下に `BOOT_ORDER=0xf14` と設定されているか確認します。設定されていない場合は設定中に何かこじれているので、1〜3を繰り返します。
* Ref: https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#configuration-properties
![](2023-01-29-18-07-12.png)

### 5. USBにOSを焼いて挿し、再起動

* SDカードを抜き、USBにOSを焼いて再起動します。
* おお！起動します。グッド！
