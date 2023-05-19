---
title: "라즈베리 파이 + Ubuntu 22.04에 K3S 설치해서 쓰기"
description: 
date: 2023-01-18T11:56:02Z
image: cover.png
math: 
license: 
categories:
    - K8S
    - K3S
    - 라즈베리 파이
tags:
    - K8S
    - K3S
    - 라즈베리 파이
---

요즘 되게 핫하면서도 어려운 기술이 쿠버네티스인 것 같습니다.

저는 학생때부터 나름 꽤 오래(?) 홈 서버를 운영했었는데, 운영을 하면서 꽤 많은 힘든 일을 마주했습니다.

이사를 가서 네트워크를 다시 짜야 되거나, 디스크 하나가 죽어버리거나 하는 경우도 간간히 있고, 그럴 때마다 Git이랑 다른 컴퓨터를 뒤져서 사이트를 살리는게 여간.. 귀찮은 일이 아닙니다.

뭔가 내 서버가 어떻게 돌아가고 있는지 적어놓고, 언제든지 마음대로 컴퓨터를 추가하고, 디스크를 추가하고 하면 얼마나 좋을까? 하는 차에 K8S는 정말 매력적인 선택지였습니다.

그래서 K8S 공부를 해 봤는데, 보통 Vagrant 등을 이용해서 짱짱한 컴퓨터 하나에 가상 노드 하나를 놔두고, 가상 VM을 띄워서 실습을 하게 됩니다.

`근데 문제는 그 다음입니다.`

데스크탑 홈 서버를 여러대를 사서 K8S를 돌릴 수는 없으니, `가격 쌈 && 저전력 PC`를 찾게 되고, 보통 라즈베리 파이로 귀결됩니다.

그런데 강사 선생님이 이쁘게 깔아놓은 설치 스크립트만 쓰다가, ARM에 직접 K8S 깔려면 머리가 터집니다. (사실 저는 터졌습니다. 다른사람은 잘 모르겠고..)

[KubeAdm](https://github.com/kubernetes/kubeadm), [MicroK8S](https://github.com/canonical/microk8s) 를 몇번의 삽질 끝에 설치 실패했는데, K3S는 되게 간편하게 설치 되어서 경험을 공유하려 합니다.

![](image1.png)

---

`아래 환경은 Raspberry pi 4B + Ubuntu Server 22.04에서 진행했습니다.`

### 1. Master Node 설치하기

* K3S [공식 문서](https://docs.k3s.io/quick-start)를 보면 `curl -sfL https://get.k3s.io | sh -` 만 실행하면 알잘딱깔센으로 설치해준다고 합니다!
* bash에 `curl -sfL https://get.k3s.io | sh -` 를 입력해서 설치를 해 봅니다.
* 와! 설치가 잘 됩니다.
* kubectl도 같이 설치해준다고 합니다. `kubectl get nodes`를 입력해 봅니다.
* 와! 노드도 잘 뜹니다.

### 근데 사실 함정입니다.

* `kubectl get nodes`를
* 어? 갑자기 응답이 느립니다.
* `The connection to the server 127.0.0.1:6443 was refused - did you specify the right host or port?` 라는 이슈를 만납니다.
* 아... 얘도 함정인가? 불안합니다.
* 정말 다행이도 얘는 Known Issue이고, 해결 방법도 있습니다. (관련 [Github Issue](https://github.com/k3s-io/k3s/issues/4234))
* `sudo apt install linux-modules-extra-raspi && reboot`를 입력해서 추가로 필요한 의존성을 깔아주고 재부팅 한 후 `kubectl get nodes`를 하면 잘 됩니다. (공식 문서 [Link](https://docs.k3s.io/advanced#raspberry-pi))

### 2. 노드 추가하기

* Master Node를 설정했으니, Worker Node를 붙여봅시다.
* `kubectl get nodes` 를 입력하여, Master Node의 이름(Name) 을 확인합니다.
* `sudo kubectl get node <master node 이름> -ojsonpath="{.status.addresses[0].address}"` 를 입력하여, 나오는 IP 값을 확인합니다. (1)
* `sudo cat /var/lib/rancher/k3s/server/node-token` 을 입력하여, 나오는 Token값을 확인합니다. (2)

* 새 Worker Node에 접속해서 `sudo apt install linux-modules-extra-raspi && reboot` 하여 관련 의존성을 깔아줍니다.
* 이후 `curl -sfL https://get.k3s.io | K3S_URL=https://<(1)에서 확인한 ip>:6443 K3S_TOKEN=<2에서 확인한 토큰> sh -` 을 실행합니다.
* 이후 Master Node에서 `kubectl get nodes` 입력하여 정상적으로 클러스터에 Join되었는지 확인합니다.

* 추가 : Worker Node 상태가 NotReady에서 안 넘어가는 경우가 있는데, 이 경우를 대응해 봅시다.
    *  `kubectl get nodes` 입력하여 Worker Node의 이름을 기억합니다.
    *  `sudo kubectl describe node <worker node 이름> -n=kube-system` 을 입력하여 현재 노드의 상태를 확인합니다.
    *  저의 경우는 `invalid capacity 0 on image filesystem` 이라는 오류가 계속 발생해서 Worker Node가 조인이 안 되는 상황이었는데, 재부팅 하니 해결됐습니다 (..)

끝!