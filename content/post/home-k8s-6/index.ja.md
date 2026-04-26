---
title: "自宅でRaspberry Piクラスタを使ってデータセンターを構築する - 6. 分散ストレージの追加とLonghornでの提供"
slug: 615befc538a81902
date: 2023-12-09T10:48:57+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - Raspberry Pi
tags:
    - K8S
    - K3S
    - Raspberry Pi
    - Longhorn
    - ストレージ
    - ディスクマウント
---

_注：筆者は韓国在住のため、本文には韓国特有の文脈が含まれることがあります。_

### 1. 新規ディスクの追加とマウント

- この内容は一般的なSSD / HDDのマウントと同じです！うまくいかない場合は、`Ubuntu ハードディスク マウント`などで検索して進めても問題ありません。

1. SSDまたはHDDをRaspberry Piに接続します。

2. `sudo fdisk -l`コマンドを実行して、マウントしたハードディスクの名前を確認します。私の場合は`/dev/sda`です。

![マウントされたハードディスクの内容](image.png)

3. `sudo wipefs -a /dev/<先ほど確認したディスク>`を実行し、注意してディスクをフォーマットします。（例：`sudo wipefs -a /dev/sda`）
4. `sudo mkfs.ext4 /dev/<先ほど確認したディスク>`を実行し、ディスクをext4フォーマットにします。
5. `sudo blkid -s UUID -o value /dev/<先ほど確認したディスク>`を実行し、ディスクの固有ID（680dfccb-9d8f-431c-ab3b-8c1e6c86e04fの形式）を取得します。
6. `sudo mkdir /storage-ssd`（場所は自由です。私は便宜上、ルートに`/storage-ssd`というフォルダを作成してマウントしました。）でマウント用のフォルダを作成します。
7. sudo権限で`/etc/fstab`を開き、最後の行に`UUID=7cf3fc21-74d6-4c01-a835-f8bd36bc3f7b /storage-ssd ext4 defaults 0 0`を追加します。（6で作成したフォルダにマウント）
8. `sudo mount -a`で、先ほど登録したディスクをマウントします。

### 2. 新規ディスクをLonghornに登録

それでは、新しいディスクをLonghornに登録してみましょう！

1. 先ほど設定したロードバランサのIP（`http://192.168.0.201/`）をブラウザで開きます。
2. Node -> 該当Nodeを選択 -> Edit Node and disksを選択します。

![Longhorn設定スクリーンショット、Edit Node and disks](image-1.png)

3. disk tagに、StorageClass設定時の`diskSelector`と同じタグを入力し、このディスクがそのStorageClassに属することを示します。次にPathを上で設定したマウントフォルダに指定します。
![Longhorn設定スクリーンショット](image-2.png)

4. その後、保存を押し、ボリュームが正しく認識されているか確認します。


### 3. おわりに

お疲れ様でした！これでLonghornを使って、今後のアプリケーションに安定的に分散ストレージシステムを提供できるようになりました！

次は、Sealed Secretsを使ってパスワードなどの機密情報をGitで管理する方法を紹介し、その後Private Docker Registryを構築する予定です。

ありがとうございました！
