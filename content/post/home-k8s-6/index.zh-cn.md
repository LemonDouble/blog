---
title: "在家用树莓派集群搭建数据中心 - 6. 添加分布式存储并通过 Longhorn 提供服务"
slug: 615befc538a81902
date: 2023-12-09T10:48:57+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - 树莓派
tags:
    - K8S
    - K3S
    - 树莓派
    - Longhorn
    - 存储
    - 磁盘挂载
---

_注：作者居住在韩国，部分内容包含韩国特有的背景。_

### 1. 添加并挂载新磁盘

- 这部分内容与一般的 SSD / HDD 挂载相同！如果遇到问题，可以搜索 `Ubuntu 硬盘挂载` 等关键词参考操作。

1. 将 SSD 或 HDD 连接到树莓派。

2. 输入 `sudo fdisk -l` 命令，查找已挂载硬盘的名称。我这里是 `/dev/sda`。

![挂载的硬盘内容](image.png)

3. 输入 `sudo wipefs -a /dev/<刚才确认的磁盘>`，谨慎地格式化磁盘。（示例：`sudo wipefs -a /dev/sda`）
4. 输入 `sudo mkfs.ext4 /dev/<刚才确认的磁盘>`，将磁盘格式化为 ext4 格式。
5. 输入 `sudo blkid -s UUID -o value /dev/<刚才确认的磁盘>`，获取该磁盘的唯一 ID（形如 680dfccb-9d8f-431c-ab3b-8c1e6c86e04f）。
6. 用 `sudo mkdir /storage-ssd`（位置随意，我为方便起见挂载到了根目录下名为 `/storage-ssd` 的文件夹）创建挂载用的文件夹。
7. 用 sudo 权限打开 `/etc/fstab`，在最后一行添加 `UUID=7cf3fc21-74d6-4c01-a835-f8bd36bc3f7b /storage-ssd ext4 defaults 0 0`。（挂载到第 6 步创建的文件夹）
8. 通过 `sudo mount -a` 挂载刚刚注册的磁盘。

### 2. 在 Longhorn 中注册新磁盘

现在让我们把新磁盘注册到 Longhorn 吧！

1. 在浏览器中打开之前设置的负载均衡器 IP（`http://192.168.0.201/`）。
2. 依次选择 Node -> 对应的 Node -> Edit Node and disks。

![Longhorn 设置截图，Edit Node and disks](image-1.png)

3. 在 disk tag 中输入与设置 StorageClass 时的 `diskSelector` 相同的标签，告诉系统该磁盘属于该 StorageClass。然后将 Path 指定为上面设置的挂载文件夹。
![Longhorn 设置截图](image-2.png)

4. 之后点击保存，并确认卷是否被正常识别。


### 3. 结语

辛苦了！现在可以通过 Longhorn 为后续的应用稳定地提供分布式存储系统了！

接下来将介绍如何通过 Sealed Secrets 将密码等敏感信息用 Git 进行管理，之后还计划实现 Private Docker Registry。

谢谢！
