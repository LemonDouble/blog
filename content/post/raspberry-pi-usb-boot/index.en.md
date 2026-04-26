---
title: "After Killing Three SD Cards, Here's How I Set Up Raspberry Pi USB Boot"
slug: 977ba9d70a2a7029
date: 2023-01-29T17:31:39+09:00
image: cover.png
categories:
    - Raspberry Pi
tags:
    - Raspberry Pi
    - USB Boot
    - EEPROM
    - SD Card
---

_Note: I'm based in Korea, so some context here is Korea-specific._

The Raspberry Pi is great in every way, except SD card booting can be incredibly frustrating.

When you're tinkering, pulling SD cards in and out, scratching one and having it stop being recognized... it really gets to you.

I know SD cards and USB drives use similar technology, but...

In my experience, USB drives have generally been more reliable (in terms of physical durability).

So let's ditch the SD card and boot from USB!


`The following is written for an Ubuntu 22.04 Server environment.`

### 1. Flash the SD card

* To boot from USB, you need to tweak the bootloader, which means you have to boot from an SD card at least once.
* Flash the SD card and connect via SSH. All subsequent commands are run from the terminal.

### 2. Update the EEPROM bootloader

* EEPROM: Remember the ROM you learned about in computer architecture class? Think of this as a ROM that you can write to and erase.
* This is where the bootloader is stored. Roughly speaking, bootloader versions from around September 2020 onward support USB booting.
* Let's update it while we're at it.

Ref: https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#automaticupdates

1. Run `sudo rpi-eeprom-update` to check the current/latest bootloader version.
2. Run `sudo rpi-eeprom-update -a` to schedule the bootloader update.
3. Run `sudo reboot` to reboot, and the bootloader update will run during boot.
4. Run `sudo rpi-eeprom-update` again to verify the update went through.


- Update (2023.05.19)
  - The `rpi-eeprom-update` above is supported only on RPI4 and later.
  - If you're using RPI 3B or earlier, refer to [Booting Raspberry Pi 3 B With a USB Drive](https://www.instructables.com/Booting-Raspberry-Pi-3-B-With-a-USB-Drive/).
  - If you're using Raspbian OS, you can follow that guide as is.
  - If you're using Ubuntu, you'll need to edit `boot/firmware/config.txt` instead of config.txt!!!


### 3. Set the boot order

* Ubuntu Server doesn't include the raspi-config program, so we need to install it first.

1. Run `sudo apt install raspi-config` to install the raspi-config program.
2. Run `sudo raspi-config`, then go to Advanced Options -> Boot Order.
3. Select `USB Boot`, then run `sudo reboot` to reboot.

### 4. Verify the boot order was applied

* Run `vcgencmd bootloader_config`.
* Check that `BOOT_ORDER=0xf14` is set below. If it's not set, something got mixed up during configuration, so repeat steps 1-3.
* Ref: https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#configuration-properties
![](2023-01-29-18-07-12.png)

### 5. Flash OS to USB, plug it in, and reboot

* Remove the SD card, flash the OS to a USB drive, and reboot.
* Boom, it boots! 
