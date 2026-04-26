---
title: "How About Trying IoT with HomeAssistant?"
slug: 107dcfce6eca0dd7
date: 2023-05-21T09:40:47+09:00
image: cover.png
categories:
    - HomeAssistant
    - IoT
    - Raspberry Pi
tags:
    - HomeAssistant
    - IoT
    - Raspberry Pi
    - Zigbee
    - ESPHome
    - Automation
    - Smart Home
---

_Note: I'm based in Korea, so some context here is Korea-specific._

IoT is a pretty fun topic, but

the entry barrier is high, and it's not very well-known, so I feel like people don't really know what they can do with it.

Among the many platforms out there, I personally installed and use HomeAssistant (hereafter HASS). I'd like to talk about why such a platform is needed and what it can improve in our lives!


### 1. What is HomeAssistant? (a.k.a. HA, Hass)

It's an integration platform that lets you manage IoT devices from various companies all in one place.

Check out the [Demo](https://demo.home-assistant.io/#/lovelace/0)!

![](2023-05-21-11-10-34.png)

### 2. What can I do with it?

- If you can program, you can literally do anything!

Let me give a few examples based on your interests.

- **Interested in automation?**
    - **Example scenario: Turning off the bathroom light is too annoying**

![](2023-05-21-11-11-46.png)

![](2023-05-21-11-11-56.png)

As shown above, you can install a smart light and a motion sensor

and automate it like this!

```kotlin
// This isn't real code, just pseudocode.
// Automation 1. Auto turn on bathroom light
if(bathroom_motion_sensor.Occupancy transfer to True){
    turn_on (bathroom_light)
}

// Automation 2. Auto turn off bathroom light
if(bathroom_motion_sensor.Occupancy transfer to False && 10_minutes_have_passed){
    turn_off(bathroom_light) 
}
```

- On the actual hass screen, the automation looks like this.

![](2023-05-21-11-12-18.png)

- **Interested in AI?**
    - Example scenario: Building a voice assistant
    - But seriously, you can do anything...

![](2023-05-21-11-16-59.png)

- **Interested in DIY circuits?**
    - Try a project like [ESPHome](https://esphome.io/)!

![](2023-05-21-11-12-28.png)

- You can buy a development board with WiFi and a CPU, like the ESPxx series, for just a few dollars!

- How about a ventilation light scenario like the one below?

![](2023-05-21-11-12-59.png)


- **Tired of checking bus times every day?**

- How about building a bus notifier using a public API?

![](2023-05-21-11-17-32.png)


Beyond this, there are tons more things you can do through the HomeAssistant platform!

And since it's fully open source, you can even modify it if you need to!

How about giving fun IoT a try together?
