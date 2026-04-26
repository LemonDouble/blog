---
title: "在家用树莓派集群搭建数据中心 - 2. 集群初始化设置篇"
slug: f4efced2e8d5b1d8
date: 2023-12-03T22:20:29+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - 树莓派
tags:
    - K8S
    - K3S
    - 树莓派
    - Ubuntu
    - DHCP
    - Supabase
    - PostgreSQL
    - 集群搭建
---

_注：作者居住在韩国，部分内容包含韩国特有的背景。_

### 硬件

![硬件构成示意图](image.png)

请参考上图来组装硬件。

如果你家里有在用电脑，那大概率已经有路由器和常用的台式机了，所以只需要在上图中加上绿色部分即可。

如果你在网上买了交换机（非网管交换机），只需把所有树莓派的网线都接到交换机上，再从路由器拉一条网线接到交换机上即可。

### 树莓派设置

参考我之前发的文章 [自己留着看的树莓派初始化设置](https://lemondouble.github.io/zh-cn/p/%EB%82%B4%EA%B0%80-%EB%B3%B4%EB%A0%A4%EA%B3%A0-%EC%98%AC%EB%A6%AC%EB%8A%94-%EB%9D%BC%EC%A6%88%EB%B2%A0%EB%A6%AC-%ED%8C%8C%EC%9D%B4-%EC%B4%88%EA%B8%B0-%EC%84%B8%ED%8C%85/) 和 [烧坏三张SD卡后气急败坏写下的树莓派 USB Boot 设置方法](https://lemondouble.github.io/zh-cn/p/sd%EC%B9%B4%EB%93%9C-%EC%84%B8%EA%B0%9C-%EB%82%A0%EB%A0%A4%EB%A8%B9%EA%B3%A0-%ED%99%94%EB%82%98%EC%84%9C-%EC%93%B0%EB%8A%94-%EB%9D%BC%EC%A6%88%EB%B2%A0%EB%A6%AC%ED%8C%8C%EC%9D%B4-usb-boot-%EC%84%B8%ED%8C%85-%EB%B0%A9%EB%B2%95/)，或其他博客的文章，在树莓派上安装 Ubuntu Server 22.04。

用 SD 卡也能搭建集群，但**强烈建议**配置 USB Boot 后从 USB 启动。

到这一步，假设你已经在树莓派上安装好 Ubuntu 并且能 SSH 连接。

之后执行 `sudo apt update -y && sudo apt upgrade -y && sudo apt install linux-modules-extra-raspi` 安装所需依赖。（参考 [在树莓派 + Ubuntu 22.04 上安装并使用 K3S](https://lemondouble.github.io/zh-cn/p/%EB%9D%BC%EC%A6%88%EB%B2%A0%EB%A6%AC-%ED%8C%8C%EC%9D%B4--ubuntu-22.04%EC%97%90-k3s-%EC%84%A4%EC%B9%98%ED%95%B4%EC%84%9C-%EC%93%B0%EA%B8%B0/)）

### DHCP 服务器设置

之后，第一次开机所有的树莓派后，每个树莓派会被分配一个内网 IP。（如果没改设置，会是 192.168.0.X 的形式）

如果不另作设置，这个内网地址会在每次开机时**从空闲 IP 中**取一个来分配，IP 会动态变化，导致每次都得在路由器管理页面里查节点 IP，非常麻烦。

~~（准确说，可能变也可能不变。多数情况会保持原样，但保险起见还是先设置好。）~~

![ipTIME DHCP 服务器设置](image-1.png)

如上图所示，找到类似 DHCP 服务器设置（这里以 ipTIME[^1] 为例）的菜单，输入第一次连接的树莓派的 **MAC 地址**（1A-2B-3C...），并为它分配一个固定 IP。

我为了方便把 192.168.0.20 设为 Control Node，192.168.0.21~30 设为 Worker Node，IP 你按自己习惯设置即可。

之后 Windows 用户用 Putty，Linux 或 Mac 用户用 ssh CLI 准备连接。

### 注册 Supabase 并获取 Postgres 连接权限

为 Control Node 分配三台太浪费资源，只用一台又怕 Control Node 挂掉时不好恢复，作为折中方案，

我们打算把状态存放在外部 SaaS 上，这样即使 Control Node 挂掉，只要更换 Control Node 就能从外部 Storage 拿回状态继续使用。

为此使用提供免费 DB 的 **Supabase**。

![500MB 以内免费！](image-2.png)

到 [Supabase](https://supabase.com/) 注册账号，配置好 Postgres 连接权限并取出连接信息。

如果可以，Supabase 也支持 PgBouncer，建议一起配置上。

（由于并发连接数限制，后面装 Portainer 时虽然能正常运行，但不用 PgBouncer 的话会冒出大量错误信息。）

### 配置 Master Node

*这里假设已经按上面步骤安装好了 Ubuntu。*

准备以下内容。

1. 用作 Master Node 的树莓派的 IP（上面 DHCP 服务器设置里分配的 IP）
   - 例如：`192.168.0.20`，**后面 Worker Node 加入时需要用到，请务必记下来。**
2. 准备 Kubernetes 集群相互 Join 时使用的随机 Token。我用了不含特殊字符、由大小写字母+数字组成的 60 位随机字符串。**后面 Worker Node 加入时需要，请务必记下来。**
   - 例如：`jUofPu4hMegXLDKgCn6FbsfWcL8mFQu3DYEiyDjQgh8447cMAqwfgYwTeNU4`
3. 准备好上面的 Postgres 连接地址。
   - 例如：`postgres://postgres:postgresPassword@db.mydb.supabase.co:5432/postgres`

```sh
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644 --disable servicelb --token <2,myToken> --node-taint CriticalAddonsOnly=true:NoExecute --bind-address <1,myControlNodeIP>  --disable local-storage --datastore-endpoint <3, postGresEndpoint>
```

按 1、2、3 替换成对应内容。下面是示例。
```sh
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644 --disable servicelb --token mytoken --node-taint CriticalAddonsOnly=true:NoExecute --bind-address 192.168.0.20  --disable local-storage --datastore-endpoint postgres://postgres:postgresPassword@db.mydb.supabase.co:5432/postgres
```

逐项看一下选项：

1. `curl -sfL https://get.k3s.io | sh -s -`：安装 K3S。
2. `--write-kubeconfig-mode 644`：把 KubeConfig 文件权限设为 644，后续配置会用到。
3. `--disable servicelb`：禁用 K3S 默认的 [Service Load Balancer](https://docs.k3s.io/networking#service-load-balancer)（klipper-lb）。后面会单独装一个 MetalLB 作为负载均衡器。
4. `--token <mytoken>`：设置 K3S 集群相互 Join 时使用的 Token。
5. `--node-taint CriticalAddonsOnly=true:NoExecute`：给 Control Node 加 Taint。也就是说，**为了稳定性，除了真正关键的容器外，其他容器不会运行在 Control Node 上。** 如果你只有一台节点，或者就想在 Control Node 上也跑容器，请去掉这个选项。
6. `--bind-address <myControlNodeIP>`：把 master node 绑定到指定 IP 的参数。
7. `--disable local-storage`：禁用 K3S 默认的 Local Storage。后面会单独装 Longhorn Storage。
8. `--datastore-endpoint <postGresEndpoint>`：用外部数据库代替内置 Etcd 作为状态存储。

在 Master Node 上执行上面的命令，Kubernetes 就开始安装了。

**接下来，为了方便修改 hosts 文件。**

执行 `sudo vi /etc/hosts`（用什么文本编辑器都可以），添加如下几行。

```sh
192.168.0.20 control01 control01.local

192.168.0.21 worker01 worker01.local
192.168.0.22 worker02 worker02.local
192.168.0.23 worker03 worker03.local
```

我这里把 192.168.0.20 取名为 control01，21 取名 worker01，22 取名 worker02。

这样设置之后，以后从 control Node SSH 连接时，直接 `ssh worker01` 就能连上，省事很多。

**最后，为了方便 Master Node 访问 Worker Node，配置一下 SSH。**

输入 `ssh-keygen` 命令，然后一路回车不输入任何内容，生成 ssh key。
之后把 `~/.ssh/id_rsa.pub` 文件内容好好复制下来。

### 配置 Worker Node

*这里假设已经按上面步骤安装好了 Ubuntu。*


**先来配置 SSH 连接。**

在 `~/.ssh/authorized_keys` 文件里换一行，把刚才复制的 id_rsa.pub 的内容粘贴进去。

之后在 control Node 上输入 `ssh worker01` 等命令，确认能正常连接。

如果连不上，可以搜索 ssh authorized_keys 注册等关键词来排查。

**接下来安装 K3S。**

准备以下内容。

1. 上面设置的 Master Node 的 IP
2. 上面设置的 Master Node 的 Token

之后执行下面的命令把 Worker Node Join 进集群。

```sh
curl -sfL https://get.k3s.io | K3S_URL=https://<1,myControlNodeIP>:6443 K3S_TOKEN=<2,myToken> sh -
```

按 1、2 替换。下面是示例。

```sh
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.0.20:6443 K3S_TOKEN=mytoken sh -
```

之后确认 Node 已经 Join。

### 确认节点 Join 并创建 Label

在 Control Node 上执行 `kubectl get nodes`，确认节点正常 Join。

接下来为了修改 ROLES 列，在 Control Node 上对每个节点执行下面的命令。

如果节点少就执行少几条，多就多执行几条。

```sh
kubectl label nodes worker01 kubernetes.io/role=worker
kubectl label nodes worker02 kubernetes.io/role=worker
kubectl label nodes worker03 kubernetes.io/role=worker
```

之后，再添加用于让创建容器时只调度到特定 Pod（Worker Node）的 Label。（上面和下面这两组都要执行！）
```sh
kubectl label nodes worker01 node-type=worker
kubectl label nodes worker02 node-type=worker
kubectl label nodes worker03 node-type=worker
```

最后做一次最终确认。

下面是我这边的输出。你的 ROLES 列可能还和我不一样。（`kubectl get nodes`）

```sh
NAME           STATUS   ROLES                  AGE    VERSION
control01      Ready    control-plane,master   307d   v1.27.4+k3s1
worker02       Ready    worker                 307d   v1.27.4+k3s1
worker01       Ready    worker                 307d   v1.27.4+k3s1
worker03       Ready    worker                 307d   v1.27.4+k3s1
```

另外，再确认一下添加的 label。（`kubectl get nodes --show-labels`）

有很多 label 是正常的，只要确认我们加的 `node-type=worker` 在就可以。

### 结语

恭喜！硬件和 Kubernetes 安装都顺利完成了。

设置过程比较长，如果哪里卡住了，欢迎在评论里告诉我！

[^1]: ipTIME 是韩国常见的家用路由器品牌，韩国家庭用户对其管理界面比较熟悉。同样的 DHCP 静态分配思路适用于其他品牌的路由器。
