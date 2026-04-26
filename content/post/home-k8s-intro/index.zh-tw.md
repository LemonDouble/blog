---
title: "在家用樹莓派叢集打造資料中心 - 序論"
slug: d42387934c57aa25
date: 2023-12-03T13:24:05+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - 樹莓派
tags:
    - K8S
    - K3S
    - 樹莓派
    - 叢集建置
    - 家庭實驗室
    - 架構
---

_註：筆者居住於韓國，部分內容包含韓國特有的背景。_

### 我第一次建置樹莓派叢集至今，大約已經過了300天。

契機其實並不算大。

我每天早上上班路上都會看一個叫[GeekNews](https://news.hada.io/)[^1]的新聞策展服務，其中一篇題為[Chick-Fil-A 的 Edge Computing 技術架構：Enterprise Restaurant Compute](https://news.hada.io/topic?id=8306)的文章引起了我的注意。

之後，「連Chick-Fil-A都在跑Kubernetes，身為一名伺服器工程師，我自己是不是也該有一個屬於自己管理的叢集？」——這就是一切的開始。

### 配置環境

我頂樓房間(...)的叢集大概長這樣。
![Alt text](image.png)
![Alt text](image-1.png)

整理一下配置規格：

- **Control Node**
  - Raspberry PI 4b+ 8GB Model * 1
  - Samsung MUF-AB FIT PLUS 64GB USB
- **ARM Worker + Storage Node**
  - Raspberry PI 4b+ 8GB Model + 500GB SSD(PNY CS900 500GB) * 3
  - Sandisk USB Ultra Fit USB 3.1 32GB * 3
- **GPU X86 Worker Node (For ML)**
  - Ryzen 5600x + 64GB DDR4 3200 RAM + RTX3090 + NVME SSD(WD Black) 1TB + WD RED Plus 4TB HDD

依以上方式配置，總共用4台樹莓派、1台X86 GPU伺服器，運行X86與ARM混合的叢集。

樹莓派的磁碟I/O為了穩定性，使用USB碟取代SD卡。

### 咦？只有1台Control Node不就無法做HA了嗎？

- 這是初次建置叢集時最大的煩惱。
  - 用3台以上配置HA雖然較安全，但考慮到Pi的運算能力並不強，我希望盡可能減少被浪費的運算資源。

- 因此得出的結論是，**Master Node只用一台**，並設置了下列安全機制。
  - 用**External Database**取代etcd作為狀態儲存。
    - 目前使用**Supabase**提供的Postgresql作為外部儲存。
  - Master Node設置了Taint，使其無法進行Job scheduling。
    - 透過這種方式，減少了在任務分配過程中master節點突然掛掉的情況。
  - Master Node使用了相對更可靠的設備(USB、散熱器等)。

儘管如此，夏天Master Node因過熱(...)USB還是出過故障，但由於所有State都存在外部DB中，10分鐘內就完成了復原。

### 目前在軟體層面已建置的內容

1. 利用K3S與External DB，建構即使Master Node當機也能恢復的系統。
2. 利用ArgoCD與Github，建構GitOps系統。
3. 利用[Mend Renovate](https://www.mend.io/renovate/)，當已安裝的Helm Chart及Private Registry有新版本發布時，自動建立PR，使叢集保持最新狀態。
4. 利用[Longhorn](https://github.com/longhorn/longhorn)實作分散式儲存，建構即使一顆SSD壞掉也能恢復的系統。
5. 建置Docker Private Registry，並利用[Docker-registry-browser](https://github.com/klausmeyer/docker-registry-browser)建構可檢視已上傳映像檔的GUI。
6. 利用[Sealed-secrets](https://github.com/bitnami-labs/sealed-secrets)，無需外部服務即可將Secret儲存於Git中進行管理。同時利用[Kubeseal-webgui](https://github.com/Jaydee94/kubeseal-webgui)，建構透過GUI便捷新增Secret的畫面。
7. 利用[kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)建置並管理監控系統。
8. 利用[CloudNativePG](https://cloudnative-pg.io/)建置HA Postgres資料庫系統，並為了防止意外，在AWS等位置建構自動備份系統。
9. 利用[Portainer](https://www.portainer.io/)建構基於Web UI的叢集管理系統。
10. 透過[MetalLB](https://metallb.universe.tf/)設定，輕鬆建構僅可在內網存取的endpoint與服務。
11. 利用[nvidia-device-plugin](https://github.com/NVIDIA/k8s-device-plugin)，讓Kubernetes上的容器能夠使用CUDA等裝置。
12. 借鑑[traefik-forward-auth](https://github.com/thomseddon/traefik-forward-auth)函式庫的構想，建構自有的認證系統。除了密碼之外，亦可透過SSO登入存取內部管理系統。

### 接下來要寫的文章

接下來我打算回顧自己建置的內容，從硬體到軟體、再到系統建構，等有時間就一篇篇地補上。
以上內容是無序排列的，寫作過程中內容可能會有變動。

此外，本建置指南是以2023年12月為基準，由我親自跟著操作並確認正常運作的方法。

### 結束序論

建置叢集時我努力尋找了相關資料，

但意外地發現這類資料並不多。
[VladoPortos](https://github.com/VladoPortos)的[Kubernetes with OpenFaaS on Raspberry Pi 4](https://rpi4cluster.com/)資料給了我很大的幫助，但國內資料不多這點確實有些可惜。


希望這份微薄的指南，能對想要開拓類似道路的朋友有所幫助。

[^1]: GeekNews是韓國的技術新聞策展網站，類似Hacker News，但面向韓國開發者社群。
