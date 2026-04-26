---
title: "Showing Off My Voice-Controlled AI Smart Home"
slug: 30f1bf25ad8b594a
date: 2024-07-26T22:07:33+09:00
image: cover.png
math: 
license: 
categories:
    - HomeAssistant
    - IoT
    - Raspberry Pi
    - AI
tags:
    - HomeAssistant
    - IoT
    - Raspberry Pi
    - Whisper
    - GPT
    - Zigbee
    - MQTT
    - Voice Recognition
    - TTS
    - Smart Home
---

_Note: I'm based in Korea, so some context here is Korea-specific._

## Preview of the smart home I built (Japanese audio warning)

- [Link](smart-home.mp4)

### 1. HomeAssistant?

It all starts with a platform called HomeAssistant, which I've [written about before](/en/p/homeassistant%EB%A1%9C-iot-%ED%95%9C%EB%B2%88-%ED%95%B4-%EB%B3%B4%EC%A7%80-%EC%95%8A%EC%9D%84%EB%9E%98%EC%9A%94/).

For HomeAssistant, 2023 was the [Year of Voice](https://www.home-assistant.io/blog/2022/12/20/year-of-voice/).

With AI led by LLMs becoming more sophisticated, it was the year I started thinking, "We can really build Jarvis now!"

![Jarvis](image.png)

### 2. So what do you actually want to say?

I just wanted to show off (...).

![Full pipeline](image-1.png)


### 3. References and tools used

[HotWordPlugin](https://play.google.com/store/apps/details?id=nl.jolanrensen.hotwordPluginFree&hl=ko) - Wake word detection on Android

[Tasker](https://play.google.com/store/apps/details?id=net.dinglisch.android.taskerm&hl=ko) - Detects HotwordPlugin events on Android and launches a specific app

[Whisper](https://openai.com/index/introducing-chatgpt-and-whisper-apis/) - Speech-to-Text engine

[extended_openai_conversation](https://github.com/jekalmin/extended_openai_conversation) - Uses information about devices in the home and calls real device control functions via function calling.

[Bert-VITS2 (TTS)](/en/p/bert-vits2%EB%A1%9C-%EB%82%98%EB%A7%8C%EC%9D%98-tts-%EB%A7%8C%EB%93%A4%EA%B8%B0/) - The character voice is cute, right?

[Zigbee2Mqtt](https://www.zigbee2mqtt.io/) - An open-source project that bridges Zigbee signals to an MQTT queue.

[SONOFF ZBDongle-P](https://ko.aliexpress.com/item/1005003758328408.html) - A dongle that lets the Raspberry Pi transmit Zigbee signals.

[Tuya IR remote](https://ko.aliexpress.com/item/1005006007728129.html) - Receives Zigbee signals and emits IR signals.
