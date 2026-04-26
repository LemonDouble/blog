---
title: "Raspberry Pi Initial Setup (For My Future Reference)"
slug: 088bdb572b7dc509
date: 2023-01-18T11:13:14Z
image: cover.png
categories:
    - Raspberry Pi
tags:
    - Raspberry Pi
    - SSH
    - Ubuntu
    - netplan
    - Initial Setup
---

_Note: I'm based in Korea, so some context here is Korea-specific._

When you run a Raspberry Pi or a home server, you end up needing to wipe and reinitialize the server surprisingly often.

Maybe something doesn't work because of a misconfiguration, or you've installed so much stuff you no longer remember what's on there.

So there's a recurring set of tasks I do every time I reset my Raspberry Pi,

and the problem is that I always forget the details by the time I have to do it again, so every fresh setup turns into another long Google session. I figured I'd just collect them in one place.


`The instructions below are based on Raspberry Pi + Ubuntu Server. The default login password is "ubuntu".`


### 1. Enable SSH access

* Lugging an HDMI cable and keyboard over to the Pi is a huge pain.
* When you're flashing the boot disk, create an empty file named `ssh` (no extension), and SSH access will be enabled at boot. (It's blocked by default.)
* Plug the Pi into your router with an Ethernet cable, find the IP from your router's admin page, and SSH in. The hostname will be `ubuntu`.

### 2. Configure SSH login

* To enable SSH key-based login, add your public key to `~/.ssh/authorized_keys`.
* If you don't have an SSH key yet, search for something like "SSH key login" on Google and follow along!

### 3. Disable password login

* If your Pi is only reachable from your internal network, it usually doesn't matter much... but if you have an SSH key, it's better security-wise to disable password login whenever possible.
* Open `/etc/ssh/sshd_config` and set `PasswordAuthentication no`.
* Open `/etc/ssh/sshd_config.d/50-cloud-init.conf` and set `PasswordAuthentication no` there too.
* Then restart the SSH server with `sudo systemctl reload sshd`.

### 4. Connect to Wi-Fi (optional)

* If you're going to keep using Ethernet, you can skip this. I personally hate the cable mess in my room, so I set up Wi-Fi.
* Open `/etc/netplan/50-cloud-init.yaml`. (Create it if it doesn't exist.)
* Copy and paste the file below. The example below uses SSID (Wi-Fi name) = `lemon`, password = `lemon1234`.

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

### 5. Set the hostname

* Not seeing what I'm currently working on in my SSH window makes me anxious...
* Run `sudo hostnamectl set-hostname <hostname>` to change the hostname to something recognizable.


### 6. Reboot

* `sudo reboot`
* When reconnecting over Wi-Fi, the internal IP may change. If you can't connect, check your router's page and update the internal IP.
