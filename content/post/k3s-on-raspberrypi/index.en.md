---
title: "Installing K3S on Raspberry Pi + Ubuntu 22.04"
slug: fe0cac9de367fc18
date: 2023-01-18T11:56:02Z
image: cover.png
categories:
    - K8S
    - K3S
    - Raspberry Pi
tags:
    - Raspberry Pi
    - K3S
    - Ubuntu
    - Installation Guide
---

_Note: I'm based in Korea, so some context here is Korea-specific._

Kubernetes seems to be one of those technologies that's super hot but also pretty hard these days.

I've been running a home server for quite a while since my student days, and I've run into plenty of headaches along the way.

Sometimes I move to a new place and have to redo my whole network setup, or a disk dies on me. Every time that happens, digging through Git and other computers to bring my site back up is... a real pain.

I kept thinking, "Wouldn't it be great if I could write down how my server is running, and freely add machines or disks whenever I want?" That's when K8S started looking like a really attractive option.

So I started studying K8S. Usually you use something like Vagrant, set up one virtual node on a beefy machine, and practice with VMs.

`But the problem comes after that.`

You can't just buy a bunch of desktop home servers to run K8S, so you end up looking for `cheap && low-power PCs`, which usually leads to Raspberry Pi.

The thing is, after only ever using neatly prepared installation scripts from instructors, trying to install K8S directly on ARM is enough to make your head explode. (Well, mine did. Not sure about others.)

After failing several times trying to install [KubeAdm](https://github.com/kubernetes/kubeadm) and [MicroK8S](https://github.com/canonical/microk8s), K3S installed pretty easily, so I wanted to share my experience.

![](image1.png)


`The setup below was done on Raspberry Pi 4B + Ubuntu Server 22.04.`

### 1. Installing the Master Node

* According to the K3S [official docs](https://docs.k3s.io/quick-start), you just need to run `curl -sfL https://get.k3s.io | sh -` and it'll install everything nicely!
* So I ran `curl -sfL https://get.k3s.io | sh -` in bash to install it.
* Wow! Installation works.
* They say kubectl gets installed too. Let me try `kubectl get nodes`.
* Wow! The node shows up just fine.

### But there's actually a catch.

* Try running `kubectl get nodes`.
* Huh? The response is suddenly slow.
* You hit this issue: `The connection to the server 127.0.0.1:6443 was refused - did you specify the right host or port?`
* Ugh... is this another trap? I'm getting nervous.
* Thankfully this is a known issue, and there's a fix. (Related [Github Issue](https://github.com/k3s-io/k3s/issues/4234))
* Run `sudo apt install linux-modules-extra-raspi && reboot` to install the additional required dependencies, reboot, and then `kubectl get nodes` works fine. (Official docs [Link](https://docs.k3s.io/advanced#raspberry-pi))

### 2. Adding Nodes

* Now that the Master Node is set up, let's attach a Worker Node.
* Run `kubectl get nodes` to check the Master Node's name.
* Run `sudo kubectl get node <master node name> -ojsonpath="{.status.addresses[0].address}"` and note the IP value that comes out. (1)
* Run `sudo cat /var/lib/rancher/k3s/server/node-token` and note the token value that comes out. (2)

* SSH into the new Worker Node and run `sudo apt install linux-modules-extra-raspi && reboot` to install the related dependencies.
* Then run `curl -sfL https://get.k3s.io | K3S_URL=https://<ip from (1)>:6443 K3S_TOKEN=<token from (2)> sh -`.
* After that, run `kubectl get nodes` on the Master Node to confirm the Worker has properly joined the cluster.

* Bonus: If your Worker Node gets stuck in NotReady, here's how to handle it.
    *  Run `kubectl get nodes` and remember the Worker Node's name.
    *  Run `sudo kubectl describe node <worker node name> -n=kube-system` to check the current state of the node.
    *  In my case, I kept getting an `invalid capacity 0 on image filesystem` error and the Worker Node wouldn't join. A reboot fixed it (...).

That's it!
