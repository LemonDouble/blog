---
title: "在家用樹莓派叢集搭建資料中心 - 3. 安裝 Helm 並使用 MetalLB 設定負載平衡器"
slug: 06a98c88458a5fb7
date: 2023-12-06T05:20:00+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - 樹莓派
tags:
    - K8S
    - K3S
    - 樹莓派
    - Helm
    - MetalLB
    - 負載平衡器
---

_註：筆者居住於韓國，部分內容包含韓國特有的背景。_

### 安裝 Helm

- 參考文章：[Installing Helm](https://helm.sh/docs/intro/install/)

依序輸入以下指令來安裝 Helm。

```sh
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

接著設定 kubeconfig 檔案。

```sh
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
chmod 600 ~/.kube/config
```

之後在 bashrc 的最底部加入下面這一行，再使用 `source ~/.bashrc` 立即套用 Shell 設定。
```sh
export KUBECONFIG=~/.kube/config
```


Helm 在 Kubernetes 中扮演類似套件管理器的角色。

舉例來說，假設我想以叢集模式安裝 Kafka。

若我要逐一將 Kafka 部署到 Kubernetes，就必須拉取 Kafka 容器映像、開啟叢集選項、設定內部網路連線……這些工作全部都要自己完成。

另外，再想像一下要升級 Kafka 版本的情況。

得在自己撰寫的 Kafka 叢集 YAML 檔中提升容器映像版本……若叢集安裝時還使用了其他資源，就要確認相容性……等等複雜的工作都在等著你。

但如果有人事先把這些工作做好了呢？

這就叫做 `Helm Chart`，我們只要輸入 `helm install kafka`，就能取得一個網路、叢集設定都已完成的 Kafka 叢集。

不過如果大家都使用同樣的 Helm Chart，那麼像我們這種(?) 只想為了實驗而啟動一台 Kafka 容器的人，和必須由數百台 Kafka 組成叢集的大公司，就只能使用相同的設定。

為了因應這種情況，Helm 與 Chart 提供了一個名為 `Values.yaml` 的額外設定檔。

也就是說，如果我只需要一台容器，那麼仔細閱讀 chart 的 `values.yaml`，再像 `kafka.container-count : 1` 那樣設定，相應的變數就會被注入，便能彈性地以自己需要的設定使用 Helm Chart。（就像伺服器的環境變數一樣！）

### 安裝 MetalLB

- 參考文章：[Installation With Helm](https://metallb.universe.tf/installation/)

現在就用 Helm 來安裝負載平衡器 MetalLB 吧！

依序輸入以下指令。若沒有特別說明，所有指令都在 `Control Node` 上輸入。

```sh
helm repo add metallb https://metallb.github.io/metallb
helm upgrade --install metallb metallb/metallb --create-namespace --namespace metallb-system --wait
```

接著建立一個名為 `ip-address-pool.yaml` 的檔案，依下列內容撰寫，並透過 `kubectl apply -f ip-address-pool.yaml` 套用。

下面內容的意思是：將 192.168.0.200~192.168.0.250 之間的 IP 分配給負載平衡器等需要 IP 的服務。

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: ipaddresspool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.0.200-192.168.0.250
---
apiVersion: metallb.io/v1beta1

kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - ipaddresspool
```

至此，如果設定正確，使用 LoadBalancer 的服務就會被分配 IP。

輸入 `kubectl get svc -n kube-system`，確認 `traefik` 的 `EXTERNAL-IP` 是否被分配到 192.168.0.xxx。

如果沒有另外設定的話，大概會被分配到 192.168.0.200。

### 路由器連接埠轉發設定

確認 Traefik 的 IP 後，我們先把（下次要做的事情中的）一項提前處理好。

找到路由器的連接埠轉發設定，把 80、443 連接埠轉發到上面確認的 Traefik IP。

![IpTime[^1] 路由器設定頁面。已對 80、443 連接埠進行連接埠轉發。](image.png)

[^1]: IpTime 是在韓國家庭中非常普及的路由器品牌。

### MetalLB 是如何運作的？

MetalLB 可以以 ARP 模式與 BGP 模式兩種方式運作，但我們使用的是 ARP 模式，所以就非常簡單地(?) 看一下 ARP 模式的運作方式。

在看圖之前，需要一點背景知識。

1. MAC 位址是寫死在硬體上的位址，自出廠起就存在，且是唯一的。
2. 路由器的連接埠轉發，作用是當外部以特定連接埠發來請求時，將其轉換為內部網路的特定 IP 後再進行請求。
    - 例如，依照上面設定，把 80 連接埠、443 連接埠設定為 192.168.0.200，那麼對於 http（80 連接埠）、https（443 連接埠），就會在內部網路中連線到擁有 192.168.0.200 的節點。


**之後的網路請求流程如下。**

![外部透過 https、路由器 IP 進行存取](image-1.png)

1. 透過 `https://` 存取伺服器時，外部會請求 `443` 連接埠。

![路由器不知道 192.168.0.200，因此送出 ARP Request](image-2.png)

2. 由於上面設定的 `連接埠轉發` 配置，路由器必須把流量轉發到 `192.168.0.200`。但路由器目前並不知道 `192.168.0.200` 這個位址，因此會向內部網路中所有人廣播 `誰擁有 192.168.0.200，請回應` 這樣的請求。（ARP Request）

![擁有 Traefik 的節點做出 Response](image-3.png)

3. 於是擁有 Traefik 的節點回應 `啊是我；；`，並附上自己的 `MAC Address`。藉此路由器便知道，從外部進來的流量送到 `AA-AA-AA-AA` 即可。

![外部與 Traefik Node 之間可以通訊](image-4.png)

4. 這樣彼此就能互相轉發流量，因此可以正常通訊。

### 容易混淆的地方

那麼？就會出現一些容易混淆的地方。逐項大致整理如下：

1. **欸，那麼一個節點可以擁有多個 IP 嗎？**
    - 例如，前面的例子中，AA-AA-AA-AA 應該已經從 DHCP 伺服器分配到一個內部 IP 了。
    - 那這裡又額外分配了一個 IP？是的，沒錯！
    - `在 MetalLB 系統中，一個節點可以被分配多個內部 IP。`

2. **如果擁有 Traefik 的節點掛了會怎樣？**
    - 當然？短時間內會無法存取，處於停機狀態。
    - 不過 Traefik 會被排程到其他節點，之後再進行新的 ARP Request/Response，就會被轉發到新的 MAC Address（節點），故障也就恢復了。

3. **ARP Response 是由誰負責回應的？**
   - MetalLB 容器負責 ARP Response，並把送往該位址的流量傳遞到 Pod（這裡是 Traefik）。


### 結語

本文內容是我查閱文件後進行的獨立研究。（尤其是 MetalLB 相關部分）

如果有任何錯誤，歡迎隨時指正，非常感謝！
