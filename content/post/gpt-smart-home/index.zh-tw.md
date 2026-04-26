---
title: "炫耀一下我用語音控制的AI智慧家庭"
slug: 30f1bf25ad8b594a
date: 2024-07-26T22:07:33+09:00
image: cover.png
math: 
license: 
categories:
    - HomeAssistant
    - IoT
    - 樹莓派
    - AI
tags:
    - HomeAssistant
    - IoT
    - 樹莓派
    - Whisper
    - GPT
    - Zigbee
    - MQTT
    - 語音辨識
    - TTS
    - 智慧家庭
---

_註：筆者居住於韓國，部分內容包含韓國特有的背景。_

## 我做的智慧家庭預覽（注意：日文音訊）

- [Link](smart-home.mp4)

### 1. HomeAssistant？

故事從一個我[以前寫過文章](/zh-tw/p/homeassistant%EB%A1%9C-iot-%ED%95%9C%EB%B2%88-%ED%95%B4-%EB%B3%B4%EC%A7%80-%EC%95%8A%EC%9D%84%EB%9E%98%EC%9A%94/)的平台 HomeAssistant 開始。

對 HomeAssistant 來說，2023年是[語音之年](https://www.home-assistant.io/blog/2022/12/20/year-of-voice/)。

隨著以 LLM 為首的 AI 日益成熟，那一年我開始覺得：現在真的可以做出賈維斯了！

![賈維斯](image.png)

### 2. 所以你到底想說什麼？

只是想炫耀一下而已（…）。

![整體流水線](image-1.png)


### 3. 使用 / 參考的資訊

[HotWordPlugin](https://play.google.com/store/apps/details?id=nl.jolanrensen.hotwordPluginFree&hl=ko) - 在 Android 上偵測 Wake word

[Tasker](https://play.google.com/store/apps/details?id=net.dinglisch.android.taskerm&hl=ko) - 在 Android 上偵測 HotwordPlugin 事件並啟動特定應用程式

[Whisper](https://openai.com/index/introducing-chatgpt-and-whisper-apis/) - Speech To Text 引擎

[extended_openai_conversation](https://github.com/jekalmin/extended_openai_conversation) - 利用家中裝置資訊，透過 function calling 呼叫實際控制裝置狀態的函式。

[Bert-VITS2 (TTS)](/zh-tw/p/bert-vits2%EB%A1%9C-%EB%82%98%EB%A7%8C%EC%9D%98-tts-%EB%A7%8C%EB%93%A4%EA%B8%B0/) - 角色的聲音很可愛吧？

[Zigbee2Mqtt](https://www.zigbee2mqtt.io/) - 一個把 Zigbee 訊號連接到 MQTT 佇列的開源專案。

[SONOFF ZBDongle-P](https://ko.aliexpress.com/item/1005003758328408.html) - 讓樹莓派能發送 Zigbee 訊號的 dongle。

[Tuya IR 遙控器](https://ko.aliexpress.com/item/1005006007728129.html) - 接收 Zigbee 訊號後發射 IR 訊號。
