---
title: "在树莓派 + Ubuntu 22.04 上安装 K3S"
slug: fe0cac9de367fc18
date: 2023-01-18T11:56:02Z
image: cover.png
categories:
    - K8S
    - K3S
    - 树莓派
tags:
    - 树莓派
    - K3S
    - Ubuntu
    - 安装指南
---

_注：作者居住在韩国，部分内容包含韩国特有的背景。_

最近 Kubernetes 似乎是一项很火但也很难的技术。

我从学生时代起就运行家庭服务器有相当一段时间了(?)，在运营过程中遇到了不少麻烦事。

有时候搬家要重新规划网络，有时候一块硬盘坏了，每当这种情况发生，我都得翻找 Git 和其他电脑来把站点救回来，实在是......很麻烦的事情。

"要是能把我的服务器是怎么运行的记下来，并且随时能够自由地添加电脑、添加硬盘那该多好？" 在这种想法下，K8S 是非常有吸引力的选择。

于是我学习了 K8S，通常是用 Vagrant 等工具，在一台性能强劲的电脑上放一个虚拟节点，启动虚拟机来做实习。

`但问题在那之后。`

不可能买好几台桌面级家庭服务器来跑 K8S，所以会去找`便宜 && 低功耗的 PC`，通常归结到树莓派上。

但是用惯了讲师老师漂亮地准备好的安装脚本，要在 ARM 上直接装 K8S 真的会让脑袋炸开。（事实上我就炸了。别人就不知道了...）

我在尝试 [KubeAdm](https://github.com/kubernetes/kubeadm)、[MicroK8S](https://github.com/canonical/microk8s) 经历几次折腾后都安装失败了，但 K3S 安装得相当简便，因此想分享一下这次经验。

![](image1.png)


`以下环境基于 Raspberry pi 4B + Ubuntu Server 22.04 进行。`

### 1. 安装 Master Node

* 查看 K3S [官方文档](https://docs.k3s.io/quick-start) 会发现，只要运行 `curl -sfL https://get.k3s.io | sh -` 就能搞定一切的安装！
* 在 bash 中输入 `curl -sfL https://get.k3s.io | sh -` 进行安装。
* 哇！安装成功了。
* 据说 kubectl 也会一并安装。试着输入 `kubectl get nodes`。
* 哇！节点也正常显示了。

### 但其实这是个陷阱。

* 把 `kubectl get nodes`
* 咦？响应突然变慢了。
* 遇到了 `The connection to the server 127.0.0.1:6443 was refused - did you specify the right host or port?` 这个问题。
* 啊......这也是陷阱吗？开始不安。
* 万幸的是，这是 Known Issue，并且有解决方法。（相关 [Github Issue](https://github.com/k3s-io/k3s/issues/4234)）
* 输入 `sudo apt install linux-modules-extra-raspi && reboot` 安装额外需要的依赖，重启后再 `kubectl get nodes` 就正常了。（官方文档 [Link](https://docs.k3s.io/advanced#raspberry-pi)）

### 2. 添加节点

* 既然 Master Node 已经配置好了，那就来挂上 Worker Node 吧。
* 输入 `kubectl get nodes`，确认 Master Node 的名称（Name）。
* 输入 `sudo kubectl get node <master node 名称> -ojsonpath="{.status.addresses[0].address}"`，确认输出的 IP 值。(1)
* 输入 `sudo cat /var/lib/rancher/k3s/server/node-token`，确认输出的 Token 值。(2)

* 连接到新的 Worker Node，执行 `sudo apt install linux-modules-extra-raspi && reboot` 来安装相关依赖。
* 然后执行 `curl -sfL https://get.k3s.io | K3S_URL=https://<(1)中确认的 ip>:6443 K3S_TOKEN=<(2)中确认的 token> sh -`。
* 之后在 Master Node 上输入 `kubectl get nodes`，确认是否正常加入集群。

* 补充：Worker Node 状态卡在 NotReady 不动的情况，让我们来应对这种情况。
    *  输入 `kubectl get nodes` 记下 Worker Node 的名称。
    *  输入 `sudo kubectl describe node <worker node 名称> -n=kube-system` 查看当前节点的状态。
    *  我的情况是一直出现 `invalid capacity 0 on image filesystem` 错误，导致 Worker Node 无法加入，重启后解决了 (..)

完！
