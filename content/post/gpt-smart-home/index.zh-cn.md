---
title: "炫耀一下我用语音控制的AI智能家居"
slug: 30f1bf25ad8b594a
date: 2024-07-26T22:07:33+09:00
image: cover.png
math: 
license: 
categories:
    - HomeAssistant
    - IoT
    - 树莓派
    - AI
tags:
    - HomeAssistant
    - IoT
    - 树莓派
    - Whisper
    - GPT
    - Zigbee
    - MQTT
    - 语音识别
    - TTS
    - 智能家居
---

_注：作者居住在韩国，部分内容包含韩国特有的背景。_

## 我做的智能家居预览（注意：日语音频）

- [Link](smart-home.mp4)

### 1. HomeAssistant？

故事从一个我[以前写过文章](/zh-cn/p/homeassistant%EB%A1%9C-iot-%ED%95%9C%EB%B2%88-%ED%95%B4-%EB%B3%B4%EC%A7%80-%EC%95%8A%EC%9D%84%EB%9E%98%EC%9A%94/)的平台 HomeAssistant 开始。

对 HomeAssistant 来说，2023年是[语音之年](https://www.home-assistant.io/blog/2022/12/20/year-of-voice/)。

随着以 LLM 为首的 AI 日益成熟，那一年我开始觉得：现在真的可以做出贾维斯了！

![贾维斯](image.png)

### 2. 所以你到底想说什么？

只是想炫耀一下而已（…）。

![整体流水线](image-1.png)


### 3. 使用 / 参考的信息

[HotWordPlugin](https://play.google.com/store/apps/details?id=nl.jolanrensen.hotwordPluginFree&hl=ko) - 在 Android 上检测 Wake word

[Tasker](https://play.google.com/store/apps/details?id=net.dinglisch.android.taskerm&hl=ko) - 在 Android 上检测 HotwordPlugin 事件并启动特定应用

[Whisper](https://openai.com/index/introducing-chatgpt-and-whisper-apis/) - Speech To Text 引擎

[extended_openai_conversation](https://github.com/jekalmin/extended_openai_conversation) - 利用家中设备信息，通过 function calling 调用实际控制设备状态的函数。

[Bert-VITS2 (TTS)](/zh-cn/p/bert-vits2%EB%A1%9C-%EB%82%98%EB%A7%8C%EC%9D%98-tts-%EB%A7%8C%EB%93%A4%EA%B8%B0/) - 角色的声音很可爱吧？

[Zigbee2Mqtt](https://www.zigbee2mqtt.io/) - 一个把 Zigbee 信号连接到 MQTT 队列的开源项目。

[SONOFF ZBDongle-P](https://ko.aliexpress.com/item/1005003758328408.html) - 让树莓派能发送 Zigbee 信号的 dongle。

[Tuya IR 遥控器](https://ko.aliexpress.com/item/1005006007728129.html) - 接收 Zigbee 信号后发射 IR 信号。
