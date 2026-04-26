---
title: "在家用樹莓派叢集架設資料中心 - 2. 叢集初始化設定篇"
slug: f4efced2e8d5b1d8
date: 2023-12-03T22:20:29+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - 樹莓派
tags:
    - K8S
    - K3S
    - 樹莓派
    - Ubuntu
    - DHCP
    - Supabase
    - PostgreSQL
    - 叢集架設
---

_註：筆者居住於韓國，部分內容包含韓國特有的背景。_

### 硬體

![硬體構成示意圖](image.png)

請參考上圖來組裝硬體。

如果家裡有在用電腦，那大概率已經有路由器跟平常使用的桌機了，所以只需要在上圖中加上綠色部分即可。

如果你在網路上買了交換器（非網管型 Switch），只要把所有樹莓派的網路線都接到交換器上，再從分享器拉一條網路線接到交換器上即可。

### 樹莓派設定

參考我之前發布的文章 [自己留著看的樹莓派初始設定](https://lemondouble.github.io/zh-tw/p/%EB%82%B4%EA%B0%80-%EB%B3%B4%EB%A0%A4%EA%B3%A0-%EC%98%AC%EB%A6%AC%EB%8A%94-%EB%9D%BC%EC%A6%88%EB%B2%A0%EB%A6%AC-%ED%8C%8C%EC%9D%B4-%EC%B4%88%EA%B8%B0-%EC%84%B8%ED%8C%85/) 與 [燒掉三張 SD 卡後氣得寫下的樹莓派 USB Boot 設定方法](https://lemondouble.github.io/zh-tw/p/sd%EC%B9%B4%EB%93%9C-%EC%84%B8%EA%B0%9C-%EB%82%A0%EB%A0%A4%EB%A8%B9%EA%B3%A0-%ED%99%94%EB%82%98%EC%84%9C-%EC%93%B0%EB%8A%94-%EB%9D%BC%EC%A6%88%EB%B2%A0%EB%A6%AC%ED%8C%8C%EC%9D%B4-usb-boot-%EC%84%B8%ED%8C%85-%EB%B0%A9%EB%B2%95/)，或其他部落格的文章，在樹莓派上安裝 Ubuntu Server 22.04。

雖然用 SD 卡也能架設叢集，但**強烈建議**設定 USB Boot 後從 USB 開機。

到這一步，假設你已經在樹莓派上安裝好 Ubuntu 並能透過 SSH 連線。

之後執行 `sudo apt update -y && sudo apt upgrade -y && sudo apt install linux-modules-extra-raspi` 安裝所需的相依套件。（請參考 [在樹莓派 + Ubuntu 22.04 上安裝並使用 K3S](https://lemondouble.github.io/zh-tw/p/%EB%9D%BC%EC%A6%88%EB%B2%A0%EB%A6%AC-%ED%8C%8C%EC%9D%B4--ubuntu-22.04%EC%97%90-k3s-%EC%84%A4%EC%B9%98%ED%95%B4%EC%84%9C-%EC%93%B0%EA%B8%B0/)）

### DHCP 伺服器設定

之後，第一次把所有樹莓派開機後，每台樹莓派都會被分配一個內網 IP。（若沒改設定，通常為 192.168.0.X 的形式）

如果不另外設定，這個內網位址會在每次開機時**從空閒 IP 中**取一個來指定，IP 會浮動變化，導致每次都得到分享器管理頁面查節點 IP，相當麻煩。

~~（精確來說，可能會變也可能不會。多數情況會維持原樣，但為了保險先設定起來。）~~

![ipTIME DHCP 伺服器設定](image-1.png)

如上圖，找到類似 DHCP 伺服器設定的選單（這裡以 ipTIME[^1] 為例），輸入第一次連線的樹莓派的 **MAC Address**（1A-2B-3C...），然後為它指派一個固定 IP。

我為了方便把 192.168.0.20 設為 Control Node、192.168.0.21~30 設為 Worker Node，IP 你可以自由設定。

之後 Windows 使用者用 Putty，Linux 或 Mac 使用者用 ssh CLI 準備連線。

### 註冊 Supabase 並取得 Postgres 連線權限

把三台都拿來當 Control Node 太浪費資源，只用一台又怕 Control Node 掛掉時難以復原，作為折衷方案，

我們會把狀態存放在外部 SaaS 上，這樣即使 Control Node 掛掉，只要替換 Control Node 就能從外部 Storage 取回狀態繼續使用。

為此採用提供免費 DB 的 **Supabase**。

![500MB 以內免費！](image-2.png)

到 [Supabase](https://supabase.com/) 註冊帳號，設定好 Postgres 連線權限後取得連線資訊。

可以的話，Supabase 也支援 PgBouncer，建議連 PgBouncer 一起設定。

（由於有同時連線數限制，之後安裝 Portainer 時雖然能正常運作，但若沒有 PgBouncer 會冒出許多錯誤訊息。）

### 設定 Master Node

*以下假設已依上述步驟完成 Ubuntu 安裝。*

請準備以下項目。

1. 要當作 Master Node 的樹莓派 IP（上面 DHCP 伺服器設定中指派的 IP）
   - 例如：`192.168.0.20`，**之後 Worker Node 加入時會用到，請務必記下來。**
2. 準備 Kubernetes 叢集互相 Join 時使用的隨機 Token。我使用了不含特殊符號、由英文大小寫＋數字組成的 60 位隨機字串。**之後 Worker Node 加入時會用到，請務必記下來。**
   - 例如：`jUofPu4hMegXLDKgCn6FbsfWcL8mFQu3DYEiyDjQgh8447cMAqwfgYwTeNU4`
3. 準備上面取得的 Postgres endpoint。
   - 例如：`postgres://postgres:postgresPassword@db.mydb.supabase.co:5432/postgres`

```sh
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644 --disable servicelb --token <2,myToken> --node-taint CriticalAddonsOnly=true:NoExecute --bind-address <1,myControlNodeIP>  --disable local-storage --datastore-endpoint <3, postGresEndpoint>
```

依照 1、2、3 替換成實際的內容。下面是範例。
```sh
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644 --disable servicelb --token mytoken --node-taint CriticalAddonsOnly=true:NoExecute --bind-address 192.168.0.20  --disable local-storage --datastore-endpoint postgres://postgres:postgresPassword@db.mydb.supabase.co:5432/postgres
```

逐項說明選項：

1. `curl -sfL https://get.k3s.io | sh -s -`：安裝 K3S。
2. `--write-kubeconfig-mode 644`：將 KubeConfig 檔案的存取權限設為 644，之後設定會用到。
3. `--disable servicelb`：停用 K3S 預設的 [Service Load Balancer](https://docs.k3s.io/networking#service-load-balancer)（klipper-lb）。之後會另外安裝 MetalLB 作為負載平衡器。
4. `--token <mytoken>`：設定 K3S 叢集互相 Join 時使用的 Token。
5. `--node-taint CriticalAddonsOnly=true:NoExecute`：對 Control Node 加上 Taint。也就是說，**為了穩定性，除了真正重要的容器之外，其他容器都不會在 Control Node 上執行。** 如果你只有一台節點，或希望 Control Node 也能跑容器，請拿掉這個選項。
6. `--bind-address <myControlNodeIP>`：將 master node 綁定到特定 IP 的參數。
7. `--disable local-storage`：停用 K3S 預設的 Local Storage。之後會另外安裝 Longhorn Storage。
8. `--datastore-endpoint <postGresEndpoint>`：以外部 Database 取代內建 Etcd 作為狀態儲存。

在 Master Node 上執行上述指令，Kubernetes 就會開始安裝。

**接下來為了方便，修改 hosts 檔。**

執行 `sudo vi /etc/hosts`（任何文字編輯器都可以），加入如下幾行。

```sh
192.168.0.20 control01 control01.local

192.168.0.21 worker01 worker01.local
192.168.0.22 worker02 worker02.local
192.168.0.23 worker03 worker03.local
```

我這邊把 192.168.0.20 取名為 control01、21 取名 worker01、22 取名 worker02。

這樣設定後，之後從 control Node SSH 連線時，直接輸入 `ssh worker01` 就能連上，相當方便。

**最後，為了讓 Master Node 容易連到 Worker Node，設定一下 SSH。**

輸入 `ssh-keygen`，然後一路按 Enter 不輸入任何內容，產生 ssh key。
之後將 `~/.ssh/id_rsa.pub` 的內容好好複製下來。

### 設定 Worker Node

*以下假設已依上述步驟完成 Ubuntu 安裝。*


**先設定 SSH 連線。**

在 `~/.ssh/authorized_keys` 換行後，把剛才複製的 id_rsa.pub 內容貼上。

之後在 control Node 上輸入 `ssh worker01` 等指令，確認能順利連線。

若無法連線，可搜尋 ssh authorized_keys 註冊等關鍵字來排除問題。

**接著安裝 K3S。**

請準備以下項目。

1. 上面設定的 Master Node IP
2. 上面設定的 Master Node Token

接著執行下列指令，把 Worker Node Join 進來。

```sh
curl -sfL https://get.k3s.io | K3S_URL=https://<1,myControlNodeIP>:6443 K3S_TOKEN=<2,myToken> sh -
```

依 1、2 替換成實際內容。下面是範例。

```sh
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.0.20:6443 K3S_TOKEN=mytoken sh -
```

之後確認 Node 已成功 Join。

### 確認節點 Join 並建立 Label

在 Control Node 上執行 `kubectl get nodes`，確認節點都正常 Join。

接著為了修正 ROLES 欄位，在 Control Node 上對每個節點執行下面的指令。

節點少就少執行幾行，節點多就多執行幾行。

```sh
kubectl label nodes worker01 kubernetes.io/role=worker
kubectl label nodes worker02 kubernetes.io/role=worker
kubectl label nodes worker03 kubernetes.io/role=worker
```

接著加上讓建立容器時只會排程到特定 Pod（Worker Node）的 Label。（上下兩組都要執行！）
```sh
kubectl label nodes worker01 node-type=worker
kubectl label nodes worker02 node-type=worker
kubectl label nodes worker03 node-type=worker
```

最後做一次最終確認。

下面是我這邊的輸出。你的 ROLES 欄位可能還跟我不一樣。（`kubectl get nodes`）

```sh
NAME           STATUS   ROLES                  AGE    VERSION
control01      Ready    control-plane,master   307d   v1.27.4+k3s1
worker02       Ready    worker                 307d   v1.27.4+k3s1
worker01       Ready    worker                 307d   v1.27.4+k3s1
worker03       Ready    worker                 307d   v1.27.4+k3s1
```

另外也確認一下加上的 label（`kubectl get nodes --show-labels`）。

label 很多是正常的，只要確認我們加上的 `node-type=worker` 有出現就行。

### 結語

恭喜！硬體與 Kubernetes 安裝都順利完成了。

整個設定流程偏長，如果有卡關的地方，歡迎在留言區告訴我！

[^1]: ipTIME 是韓國常見的家用路由器品牌，韓國家庭使用者對其管理介面相當熟悉。同樣的 DHCP 靜態指派觀念也適用於其他品牌的路由器。
