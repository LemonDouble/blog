---
title: "내가 보려고 올리는 라즈베리 파이 초기 세팅"
slug: 088bdb572b7dc509
date: 2023-01-18T11:13:14Z
image: cover.png
categories:
    - 라즈베리 파이
tags:
    - 라즈베리 파이
    - 우분투
---

라즈베리 파이나, 홈 서버를 굴리다 보면 의외로(?) 서버를 초기화해야 하는 경우가 많습니다.

세팅을 잘못 해서 뭔가 잘 안 된다거나, 뭘 깔았는지도 모르게 뭐가 잔뜩 설치되어 있는 경우 같이요.

그래서 라즈베리 파이를 초기화하면 해 줘야 하는 일련의 작업들이 있는데, 

문제가 이게 항상 까먹을 때 쯤 초기화를 해서 세팅할 때마다 한참 구글을 뒤져야 합니다. 그래서 그걸 좀 모아두려구요.

---

`아래 내용은 Raspberry PI + Ubuntu Server 기준입니다. 초기 로그인 패스워드는 ubuntu 입니다.`


### 1. SSH 접속 가능하게 하기

* HDMI 케이블 들고와서 꼽고 키보드 꼽는거 굉장히 귀찮습니다.
* 부팅 디스크 구울때 ssh (확장자 없음) 이란 빈 파일을 하나 생성해 주면, 부팅시 ssh 접근이 가능하도록 설정됩니다. (기본은 막혀있음)
* 파이를 랜선을 이용해 공유기에 꼽고, 공유기 관리 페이지 등으로 IP를 딴 다음 SSH 접속합니다. hostname은 ubuntu로 잡힙니다.

### 2. SSH 로그인 설정

* SSH 로그인이 가능하게 ~/.ssh/authorized_keys 파일에 내 공개키를 추가해 줍시다.
* SSH 키가 없다면 `SSH 키 로그인` 같은걸 구글에 쳐서 따라해 봅니다!

### 3. 패스워드 로그인 막기

* 내부망에서만 열어둔 파이라면 보통 별 상관 없지만.. SSH Key가 있다면 패스워드 로그인은 어지간하면 막아놓는게 보안상 좋습니다.
* `/etc/ssh/sshd_config` 에 들어가서 `PasswordAuthentication no` 로 설정해 줍니다.
* `/etc/ssh/sshd_config.d/50-cloud-init.conf` 에 들어가서도 `PasswordAuthentication no` 로 설정해 줍니다.
* 이후 `sudo systemctl reload sshd` 로 ssh 서버를 재시작합니다.

### 4. Wifi 연결 (선택)

* 랜선 꼽아서 쓸 거면 안 해도 됩니다. 저는 랜선 빼면 방이 지저분해져서 일단 와이파이 세팅을 합니다.
* `/etc/netplan/50-cloud-init.yaml` 파일을 엽니다. (없으면 만듭니다.)
* 아래 파일을 복사 붙여넣기 합니다. 아래 설정 파일은 SSID (와이파이 이름) = lemon, 패스워드는 lemon1234의 예시입니다.

```yaml
# This file is generated from information provided by the datasource.  Changes
# to it will not persist across an instance reboot.  To disable cloud-init's
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
    ethernets:
        eth0:
            dhcp4: true
            optional: true
    wifis:
      wlan0:
        dhcp4: true
        optional: true
        access-points:
          "lemon":
            password: "lemon1234"
    version: 2
```

### 5. Hostname 설정

* SSH 창에서 지금 뭐 건들고 있는지 안 보이면 불안합니다...
* `sudo hostnamectl set-hostname <호스트명>` 입력해서 호스트 이름도 알기 쉽게 바꿔줍니다.


### 6. 재부팅

* `sudo reboot`
* 무선랜으로 재연결시 내부 IP가 바뀔 수도 있으니, 연결 안 되면 공유기 페이지 보고 내부 ip를 수정합니다.
