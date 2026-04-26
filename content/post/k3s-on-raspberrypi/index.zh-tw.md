---
title: "在樹莓派 + Ubuntu 22.04 上安裝 K3S"
slug: fe0cac9de367fc18
date: 2023-01-18T11:56:02Z
image: cover.png
categories:
    - K8S
    - K3S
    - 樹莓派
tags:
    - 樹莓派
    - K3S
    - Ubuntu
    - 安裝指南
---

_註：筆者居住於韓國，部分內容包含韓國特有的背景。_

最近 Kubernetes 似乎是一項很熱門但也很困難的技術。

我從學生時代起就運行家庭伺服器有相當一段時間了(?)，在營運過程中遇到了不少麻煩事。

有時候搬家要重新規劃網路，有時候一顆硬碟壞了，每當這種情況發生，我都得翻找 Git 和其他電腦來把網站救回來，實在是......很麻煩的事情。

「要是能把我的伺服器是怎麼運行的記下來，並且隨時能夠自由地新增電腦、新增硬碟那該多好？」在這種想法下，K8S 是非常有吸引力的選擇。

於是我學習了 K8S，通常是用 Vagrant 等工具，在一台效能強勁的電腦上放一個虛擬節點，啟動虛擬機來做實習。

`但問題在那之後。`

不可能買好幾台桌面級家庭伺服器來跑 K8S，所以會去找`便宜 && 低功耗的 PC`，通常歸結到樹莓派上。

但是用慣了講師老師漂亮地準備好的安裝腳本，要在 ARM 上直接裝 K8S 真的會讓腦袋炸開。（事實上我就炸了。別人就不知道了...）

我在嘗試 [KubeAdm](https://github.com/kubernetes/kubeadm)、[MicroK8S](https://github.com/canonical/microk8s) 經歷幾次折騰後都安裝失敗了，但 K3S 安裝得相當簡便，因此想分享一下這次經驗。

![](image1.png)


`以下環境基於 Raspberry pi 4B + Ubuntu Server 22.04 進行。`

### 1. 安裝 Master Node

* 查看 K3S [官方文件](https://docs.k3s.io/quick-start) 會發現，只要執行 `curl -sfL https://get.k3s.io | sh -` 就能搞定一切的安裝！
* 在 bash 中輸入 `curl -sfL https://get.k3s.io | sh -` 進行安裝。
* 哇！安裝成功了。
* 據說 kubectl 也會一併安裝。試著輸入 `kubectl get nodes`。
* 哇！節點也正常顯示了。

### 但其實這是個陷阱。

* 把 `kubectl get nodes`
* 咦？回應突然變慢了。
* 遇到了 `The connection to the server 127.0.0.1:6443 was refused - did you specify the right host or port?` 這個問題。
* 啊......這也是陷阱嗎？開始不安。
* 萬幸的是，這是 Known Issue，並且有解決方法。（相關 [Github Issue](https://github.com/k3s-io/k3s/issues/4234)）
* 輸入 `sudo apt install linux-modules-extra-raspi && reboot` 安裝額外需要的相依套件，重新啟動後再 `kubectl get nodes` 就正常了。（官方文件 [Link](https://docs.k3s.io/advanced#raspberry-pi)）

### 2. 新增節點

* 既然 Master Node 已經設定好了，那就來掛上 Worker Node 吧。
* 輸入 `kubectl get nodes`，確認 Master Node 的名稱（Name）。
* 輸入 `sudo kubectl get node <master node 名稱> -ojsonpath="{.status.addresses[0].address}"`，確認輸出的 IP 值。(1)
* 輸入 `sudo cat /var/lib/rancher/k3s/server/node-token`，確認輸出的 Token 值。(2)

* 連線到新的 Worker Node，執行 `sudo apt install linux-modules-extra-raspi && reboot` 來安裝相關相依套件。
* 然後執行 `curl -sfL https://get.k3s.io | K3S_URL=https://<(1)中確認的 ip>:6443 K3S_TOKEN=<(2)中確認的 token> sh -`。
* 之後在 Master Node 上輸入 `kubectl get nodes`，確認是否正常加入叢集。

* 補充：Worker Node 狀態卡在 NotReady 不動的情況，讓我們來應對這種情況。
    *  輸入 `kubectl get nodes` 記下 Worker Node 的名稱。
    *  輸入 `sudo kubectl describe node <worker node 名稱> -n=kube-system` 查看目前節點的狀態。
    *  我的情況是一直出現 `invalid capacity 0 on image filesystem` 錯誤，導致 Worker Node 無法加入，重新啟動後解決了 (..)

結束！
