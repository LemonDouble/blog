---
title: "给自己看的树莓派初始设置"
slug: 088bdb572b7dc509
date: 2023-01-18T11:13:14Z
image: cover.png
categories:
    - 树莓派
tags:
    - 树莓派
    - SSH
    - Ubuntu
    - netplan
    - 初始设置
---

_注：作者居住在韩国，部分内容包含韩国特有的背景。_

折腾树莓派或者家庭服务器时，意外(?)地经常需要把服务器初始化重来。

可能是因为设置错了导致某些东西跑不起来，或者是装了一堆自己都不记得装过什么的东西。

于是每次重置树莓派后都有一套固定要做的工作，

但问题是每次都是隔了很久才会再初始化一次，等真要做的时候早就忘得差不多了，每次设置都得在Google里翻半天。所以干脆整理在这里。


`以下内容基于Raspberry Pi + Ubuntu Server。初始登录密码是 ubuntu。`


### 1. 开启SSH访问

* 拿HDMI线接上、再接键盘真的非常麻烦。
* 在烧录启动盘时，创建一个名为 `ssh`(无扩展名)的空文件，启动时就会开启SSH访问。(默认是关闭的)
* 用网线把树莓派接到路由器上，从路由器管理页面拿到IP后SSH连过去。hostname会是 `ubuntu`。

### 2. SSH登录设置

* 为了能用SSH密钥登录，把自己的公钥加到 `~/.ssh/authorized_keys` 文件里。
* 如果没有SSH密钥，可以在Google搜索"SSH 密钥 登录"之类的关键词跟着做！

### 3. 禁用密码登录

* 如果只在内网开放的树莓派，一般问题不大……但只要有SSH Key，从安全角度建议尽量关掉密码登录。
* 打开 `/etc/ssh/sshd_config`，设置 `PasswordAuthentication no`。
* 再打开 `/etc/ssh/sshd_config.d/50-cloud-init.conf`，同样设置 `PasswordAuthentication no`。
* 然后用 `sudo systemctl reload sshd` 重启SSH服务。

### 4. 连接Wi-Fi(可选)

* 如果一直用网线就可以跳过。我嫌网线让房间显得乱，所以会设置一下Wi-Fi。
* 打开 `/etc/netplan/50-cloud-init.yaml` 文件。(没有的话就新建。)
* 把下面的文件复制粘贴进去。下面的配置文件是SSID(Wi-Fi名称) = lemon、密码为 lemon1234 的示例。

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

### 5. 设置Hostname

* SSH窗口里看不到自己当前在操作哪台机器会让人不太安心……
* 输入 `sudo hostnamectl set-hostname <主机名>` 把hostname也改成容易辨认的名字。


### 6. 重启

* `sudo reboot`
* 通过无线网络重连时内网IP可能会变，连不上的话就看路由器页面更新一下内网IP。
