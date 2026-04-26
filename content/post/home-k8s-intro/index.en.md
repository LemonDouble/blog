---
title: "Building a Datacenter at Home with a Raspberry Pi Cluster - Introduction"
slug: d42387934c57aa25
date: 2023-12-03T13:24:05+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - Raspberry Pi
tags:
    - K8S
    - K3S
    - Raspberry Pi
    - Cluster Setup
    - Homelab
    - Architecture
---

_Note: I'm based in Korea, so some context here is Korea-specific._

### About 300 days have passed since I first built my Raspberry Pi cluster.

The trigger wasn't really that big a deal.

Every morning on my way to work, I read a news curation service called [GeekNews](https://news.hada.io/)[^1], and one day an article titled [Chick-Fil-A's Edge Computing Architecture: Enterprise Restaurant Compute](https://news.hada.io/topic?id=8306) caught my eye.

After that, the thought of "even Chick-Fil-A runs Kubernetes, so as a server engineer, shouldn't I have at least one cluster of my own to manage?" was where it all began.

### Setup

My rooftop room (...) cluster looks roughly like this.
![Alt text](image.png)
![Alt text](image-1.png)

To summarize the spec:

- **Control Node**
  - Raspberry PI 4b+ 8GB Model * 1
  - Samsung MUF-AB FIT PLUS 64GB USB
- **ARM Worker + Storage Node**
  - Raspberry PI 4b+ 8GB Model + 500GB SSD(PNY CS900 500GB) * 3
  - Sandisk USB Ultra Fit USB 3.1 32GB * 3
- **GPU X86 Worker Node (For ML)**
  - Ryzen 5600x + 64GB DDR4 3200 RAM + RTX3090 + NVME SSD(WD Black) 1TB + WD RED Plus 4TB HDD

So in total, I'm running a mixed X86/ARM cluster with 4 Raspberry Pis and 1 X86 GPU server.

For disk I/O on the Raspberry Pis, I use USB drives instead of SD cards for stability reasons.

### Wait, with only 1 Control Node, you can't do HA, right?

- This was my biggest concern when first building the cluster.
  - Configuring 3+ nodes for HA is safer, but considering the Pi's limited compute power, I wanted to minimize wasted compute as much as possible.

- As a result, I went with a **single Master Node** but added the following safeguards.
  - Instead of etcd, I use an **External Database** as the state store.
    - Currently I'm using **Supabase**'s Postgresql as the external store.
  - The Master Node has a Taint applied so it can't schedule jobs.
    - This reduces the impact if the master node suddenly dies during job allocation.
  - I'm using slightly more reliable hardware for the Master Node (USB, cooler, etc.).

Even so, there was a time during summer when the USB on the Master Node died from overheating (...), but since all the State exists in the external DB, I was able to recover within 10 minutes.

### What I've built so far on the software side

1. Using K3S and an External DB, I built a system that can recover even if the Master Node goes down.
2. Using ArgoCD and Github, I built a GitOps system.
3. Using [Mend Renovate](https://www.mend.io/renovate/), when new versions of installed Helm Charts or Private Registry images are released, PRs are automatically created to keep the cluster up to date.
4. Using [Longhorn](https://github.com/longhorn/longhorn), I implemented distributed storage so the system can recover even if a single SSD fails.
5. I built a Docker Private Registry, and using [Docker-registry-browser](https://github.com/klausmeyer/docker-registry-browser), I built a GUI to view uploaded images.
6. Using [Sealed-secrets](https://github.com/bitnami-labs/sealed-secrets), I store and manage Secrets in Git without external services. Also, using [Kubeseal-webgui](https://github.com/Jaydee94/kubeseal-webgui), I built a GUI screen for conveniently adding Secrets.
7. Using [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack), I built and manage a monitoring system.
8. Using [CloudNativePG](https://cloudnative-pg.io/), I built an HA Postgres database system, and to prevent any unforeseen accidents, I set up automatic backups to AWS, etc.
9. Using [Portainer](https://www.portainer.io/), I built a system for cluster management based on a web UI.
10. Through [MetalLB](https://metallb.universe.tf/) configuration, I can easily build endpoints and services accessible only from the internal network.
11. Using [nvidia-device-plugin](https://github.com/NVIDIA/k8s-device-plugin), I enabled containers on Kubernetes to use devices like CUDA.
12. Borrowing the idea from the [traefik-forward-auth](https://github.com/thomseddon/traefik-forward-auth) library, I built my own authentication system. Beyond passwords, I made internal management systems accessible via SSO login.

### Posts to come

Going forward, I plan to revisit what I've built and add posts one by one when I have time, covering everything from hardware to software to system configuration.
The above is listed without any particular order, and the content may change as I write.

Additionally, this build guide is based on the method I personally followed and verified to work as of December 2023.

### Wrapping up the introduction

While building the cluster, I searched hard for related materials,

but found that surprisingly, there aren't that many out there.
[VladoPortos](https://github.com/VladoPortos)'s [Kubernetes with OpenFaaS on Raspberry Pi 4](https://rpi4cluster.com/) was very helpful, but it was unfortunate that there weren't many Korean-language resources.


Even if only modestly, I hope this guide helps anyone trying to pioneer a similar path.

[^1]: GeekNews is a Korean tech news curation site, similar to Hacker News but focused on the Korean dev community.
