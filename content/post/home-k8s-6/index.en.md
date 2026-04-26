---
title: "Building a Datacenter at Home with a Raspberry Pi Cluster - 6. Adding Distributed Storage and Serving It via Longhorn"
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
    - Storage
    - Disk Mount
---

_Note: I'm based in Korea, so some context here is Korea-specific._

### 1. Add and Mount a New Disk

- This part is the same as a regular SSD / HDD mount! If something doesn't work, feel free to search for `Ubuntu hard disk mount` and follow along.

1. Connect the SSD or HDD to the Raspberry Pi.

2. Run `sudo fdisk -l` to find the name of the mounted hard disk. In my case, it's `/dev/sda`.

![Mounted hard disk contents](image.png)

3. Carefully format the disk by running `sudo wipefs -a /dev/<disk you just identified>`. (Example: `sudo wipefs -a /dev/sda`)
4. Run `sudo mkfs.ext4 /dev/<disk you just identified>` to format the disk as ext4.
5. Run `sudo blkid -s UUID -o value /dev/<disk you just identified>` to get the unique ID of the disk (in the form 680dfccb-9d8f-431c-ab3b-8c1e6c86e04f).
6. Create a folder to mount with `sudo mkdir /storage-ssd` (the location is up to you; for convenience, I mounted it to a folder called `/storage-ssd` at the root).
7. Open `/etc/fstab` with sudo privileges and add `UUID=7cf3fc21-74d6-4c01-a835-f8bd36bc3f7b /storage-ssd ext4 defaults 0 0` to the very last line. (Mount to the folder created in step 6.)
8. Run `sudo mount -a` to mount the disk you just registered.

### 2. Register the New Disk with Longhorn

Now let's register the new disk with Longhorn!

1. Open the load balancer IP you set up earlier (`http://192.168.0.201/`) in your browser.
2. Select Node -> the relevant Node -> Edit Node and disks.

![Longhorn settings screenshot, Edit Node and disks](image-1.png)

3. In the disk tag field, enter the same tag as the `diskSelector` you set when configuring the StorageClass to indicate that this disk belongs to that StorageClass. Then set the Path to the mount folder configured above.
![Longhorn settings screenshot](image-2.png)

4. Click save and verify that the volume is recognized correctly.


### 3. Wrapping Up

Great work! You can now reliably supply a distributed storage system to subsequent applications using Longhorn!

Next, I'll cover how to manage sensitive information like passwords through Git using Sealed Secrets, and after that I plan to set up a Private Docker Registry.

Thank you!
