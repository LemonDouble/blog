---
title: "집에서 라즈베리 파이 클러스터로 데이터센터 차리기 - 2. 클러스터 초기 설정 편"
slug: f4efced2e8d5b1d8
date: 2023-12-03T22:20:29+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - 라즈베리 파이
tags:
    - K8S
    - K3S
    - 라즈베리 파이
---

### 하드웨어

![하드웨어 구성 도식](image.png)

위 그림을 참고하여, 하드웨어 구성을 시켜 줍니다.

컴퓨터를 사용하는 집이라면 아마 높은 확률로 라우터와 기본 데스크탑은 설정되어 있을테니, 위 도식에서 초록색 부분만 추가하시면 됩니다.

인터넷에서 스위치허브 (Unmanaged Switch)를 구매하셨다면, 단순히 라즈베리 파이의 랜선을 전부 스위치허브에 연결하고, 공유기에서 인터넷 선을 하나 들고와서 스위치에 연결하면 됩니다.

### 라즈베리 파이 설정

이전의 제가 발행한 글인 [내가 보려고 올리는 라즈베리 파이 초기 세팅](https://lemondouble.github.io/p/%EB%82%B4%EA%B0%80-%EB%B3%B4%EB%A0%A4%EA%B3%A0-%EC%98%AC%EB%A6%AC%EB%8A%94-%EB%9D%BC%EC%A6%88%EB%B2%A0%EB%A6%AC-%ED%8C%8C%EC%9D%B4-%EC%B4%88%EA%B8%B0-%EC%84%B8%ED%8C%85/) 과, [SD카드 세개 날려먹고 화나서 쓰는 라즈베리파이 USB Boot 세팅 방법](https://lemondouble.github.io/p/sd%EC%B9%B4%EB%93%9C-%EC%84%B8%EA%B0%9C-%EB%82%A0%EB%A0%A4%EB%A8%B9%EA%B3%A0-%ED%99%94%EB%82%98%EC%84%9C-%EC%93%B0%EB%8A%94-%EB%9D%BC%EC%A6%88%EB%B2%A0%EB%A6%AC%ED%8C%8C%EC%9D%B4-usb-boot-%EC%84%B8%ED%8C%85-%EB%B0%A9%EB%B2%95/), 혹은 타 블로그의 글을 참고하여 라즈베리 파이에 우분투 서버 22.04를 설치해 줍니다.

SD카드로도 클러스터 구축은 가능하나, USB Boot 설정을 한 후 USB로 부팅하는것을 **강력하게 권장** 드립니다.

여기까지 오셨다면, 라즈베리 파이 Ubuntu 설치 + SSH 연결이 가능하다고 가정합니다.

이후, `sudo apt update -y && sudo apt upgrdae -y && sudo apt install linux-modules-extra-raspi` 로 필요한 종속성 설치를 진행합니다. ([라즈베리 파이 + Ubuntu 22.04에 K3S 설치해서 쓰기](https://lemondouble.github.io/p/%EB%9D%BC%EC%A6%88%EB%B2%A0%EB%A6%AC-%ED%8C%8C%EC%9D%B4--ubuntu-22.04%EC%97%90-k3s-%EC%84%A4%EC%B9%98%ED%95%B4%EC%84%9C-%EC%93%B0%EA%B8%B0/) 참고)

### DHCP 서버 설정

이후, 최초 한번 모든 파이를 켜 주고 나면 파이가 내부 IP를 할당받게 됩니다. (설정을 바꾸지 않았다면, 192.168.0.X 형태)

별도로 설정하지 않았다면, 이 내부 주소는 파이의 전원이 켜질 때마다 **남는 IP 중 하나** 를 가져와서 설정하게 되는데, 이 경우 IP가 유동적으로 바뀌어 매번 별도로 노드의 IP를 공유기 관리자 페이지 등에서 확인해야 하는 번거로움이 있습니다.

~~(정확히는, 바뀔 수도 안 바뀔 수도 있습니다. 대부분 그대로 유지하려고 하긴 하지만, 만일을 위해 미리 세팅해 둡니다.)~~

![IpTime DHCP 서버 설정](image-1.png)

위와 같이 DHCP 서버 설정 (Iptime 기준) 과 비슷한 메뉴를 찾아, 최초 연결된 파이의 **Mac Addr** (1A-2B-3C...) 를 입력한 후, 특정 IP를 할당해 줍시다.

저는 편의상 192.168.0.20(Control Node), 192.168.0.21~30(Worker Node)로 설정하였으나, 해당 IP는 편하신 대로 설정하면 됩니다.

이후 Windows 사용자라면 Putty를 이용해, Linux나 Mac 사용자라면 ssh CLI를 통해 연결 준비를 합니다.

### Supabase 가입 및 Postgres 접속 권한 가져오기

Control Node를 세대나 할당하기엔 자원이 너무 아깝고, 1대만 할당하자니 만약 Control Node가 죽었을 시 복구가 어려워서 선택한 차선책으로, 

저희는 만약 Control Node가 죽더라도 State를 외부 SaaS에 보관하여, 만약 Control Node가 죽더라도 Control Node를 교체하면 외부 Storage에서 상태를 가져와 사용할 수 있도록 구축할 예정입니다.

이를 위해서 무료 DB를 제공하는 **Supabase** 를 사용합니다.

![500MB까지 공짜!](image-2.png)

[Supabase](https://supabase.com/) 에 가입한 후, Postgres 접속 권한을 설정 후 가져옵니다.

설정할 수 있다면, Supabase는 PgBouncer도 지원하니, PgBouncer도 설정하면 좋습니다. 

(동시 연결 수 제한 떄문에, 이후 Portainer를 설치하면 정상 작동은 하나 PgBouncer 없이는 에러 메세지가 많이 뜹니다.)

### Matser Node 설정하기

*위 설정을 따라와 Ubuntu가 설치되어 있다고 가정합니다.*

다음을 준비합니다.

1. Mater Node로 사용할 라즈베리 파이의 IP (위 DHCP 서버 설정에서 설정한 IP)
   - 예시 : `192.168.0.20` 형태, **이후 Worker Node 연결에 필요하므로, 반드시 기록해 둡니다.**
2. 쿠버네티스 클러스터가 서로 Join할때 사용할 랜덤 토큰을 준비합니다. 저는 특문 없이 알파벳 대소문자+숫자 조합으로 60글자짜리 Random String을 사용했습니다. **이후 Worker Node 연결시 필요하므로, 반드시 기록해 둡니다.**
   - 예시 : `jUofPu4hMegXLDKgCn6FbsfWcL8mFQu3DYEiyDjQgh8447cMAqwfgYwTeNU4`
3. 위에서 준비한 Postgres 엔드포인트를 준비합니다.
   - 예시 : `postgres://postgres:postgresPassword@db.mydb.supabase.co:5432/postgres`

```sh
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644 --disable servicelb --token <2,myToken> --node-taint CriticalAddonsOnly=true:NoExecute --bind-address <1,myControlNodeIP>  --disable local-storage --datastore-endpoint <3, postGresEndpoint>
```

1,2,3에 맞춰 다음과 같이 설정합니다. 아래는 예시입니다.
```sh
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644 --disable servicelb --token mytoken --node-taint CriticalAddonsOnly=true:NoExecute --bind-address 192.168.0.20  --disable local-storage --datastore-endpoint postgres://postgres:postgresPassword@db.mydb.supabase.co:5432/postgres
```

하나하나 옵션을 까보면 다음과 같습니다.

1. `curl -sfL https://get.k3s.io | sh -s -` : K3S를 설정합니다.
2. `--write-kubeconfig-mode 644` : KubeConfig 파일의 접근 권한을 644로 설정합니다. 추후 설정에 필요합니다.
3. `--disable servicelb` : K3S의 기본 [Service Load Balancer](https://docs.k3s.io/networking#service-load-balancer) (klipper-lb) 를 비활성화합니다. 이후 MetalLB라는 로드밸런서를 별도로 깔아 사용할 예정입니다.
4. `--token <mytoken>` : K3S 클러스터가 서로 Join할때 쓰는 토큰을 설정합니다.
5. ` --node-taint CriticalAddonsOnly=true:NoExecute` : Control Node에 Taint를 추가합니다. 즉, **안정성을 위해 정말 중요한 컨테이너를 제외하곤 Control Node에서 실행하지 않도록 합니다.**. 만약 노드가 한대거나, 상관없이 Control Node에서도 컨테이너를 실행하고 싶다면 해당 옵션을 제거해 주세요.
6. `--bind-address <myControlNodeIP>` : 마스터 노드를 특정 IP에 종속시키는 플래그입니다.
7. `--disable local-storage` : K3S의 기본 Local Storage를 비활성화합니다. 이후 Longhorn Storage를 별도로 설치해 사용할 예정입니다.
8. `--datastore-endpoint <postGresEndpoint>` : 내장 Etcd 대신 외부 Database를 상태 저장소로 사용합니다.

위 커맨드를 Master Node에서 실행하면 쿠버네티스가 설치되기 시작합니다.

**이후, 편의를 위해 Hosts 파일을 수정합니다.**

`sudo vi /etc/hosts` 를 실행 후 (어떤 텍스트 편집기로 열어도 괜찮습니다., 아래와 같은 줄을 추가합니다.

```sh
192.168.0.20 control01 control01.local

192.168.0.21 worker01 worker01.local
192.168.0.22 worker02 worker02.local
192.168.0.23 worker03 worker03.local
```

저의 경우 192.168.0.20을 control01이란 닉네임으로, 21을 worker01, 22를 worker02라는 닉네임으로 사용했습니다.

이렇게 설정하면, 이후 control Node에서 ssh 접속시 `ssh worker01` 식으로 간단히 접속이 가능해집니다.

**마지막으로, Master Node에서 쉽게 Worker Node로 접근하기 위해, SSH 설정을 진행합니다.**

`ssh-keygen` 명령어를 입력하고 아무것도 치지 말고 엔터를 입력해 ssh key를 만듭니다.
이후 `~/.ssh/id_rsa.pub` 파일을 잘 복사해 둡니다.

### Worker Node 설정하기

*위 설정을 따라와 Ubuntu가 설치되어 있다고 가정합니다.*


**먼저 SSH 접속부터 설정합니다.**

`~/.ssh/authorized_keys`에, 한 줄 줄바꿈 후 방금 복사한 id_rsa.pub의 내용을 붙여넣기 합니다.

이후 control Node에서 `ssh worker01` 등을 입력하여 연결이 잘 되는지 확인합니다.

만약 연결이 잘 되지 않는다면, ssh authorized_keys 등록 등으로 검색하여 해결합니다.

**이후 K3S를 설치합니다.**

다음을 준비합니다.

1. 위에서 설정한 Master Node의 IP
2. 위에서 설정한 Master Node의 Token

이후 아래 커맨드를 입력하여 Worker Node를 Join시킵니다.

```sh
curl -sfL https://get.k3s.io | K3S_URL=https://<1,myControlNodeIP>:6443 K3S_TOKEN=<2,myToken> sh -
```

1,2에 맞춰 위와 같이 설정합니다. 아래는 예시입니다.

```sh
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.0.20:6443 K3S_TOKEN=mytoken sh -
```

이후 Node Join을 확인합니다.

### 노드 Join 확인 및 Label 생성

이후 Control Node에서 `kubectl get nodes` 를 입력하여 node들이 정상 Join한 것을 확인합니다.

이후, ROLES 부분을 수정하기 위해 Control Node에서 각 Node에 대해 아래 커맨드를 설정합니다. 

만약 Node가 적다면 적은 개수만큼, 더 많다면 더 많이 해 주시면 됩니다.

```sh
kubectl label nodes worker01 kubernetes.io/role=worker
kubectl label nodes worker02 kubernetes.io/role=worker
kubectl label nodes worker03 kubernetes.io/role=worker
```

이후, 컨테이너 생성시 특정 Pod (Worker Node) 에만 뜨도록 설정을 위한 Label을 추가합니다. (위와 아래 둘 다 해 줘야 합니다!)
```sh
kubectl label nodes worker01 node-type=worker
kubectl label nodes worker02 node-type=worker
kubectl label nodes worker03 node-type=worker
```

최종 점검을 진행합니다.

아래는 제 경우의 출력입니다. 여러분은 아직 ROLES 부분이 다를 수 있습니다. (`kubectl get nodes`)

```sh
NAME           STATUS   ROLES                  AGE    VERSION
control01      Ready    control-plane,master   307d   v1.27.4+k3s1
worker02       Ready    worker                 307d   v1.27.4+k3s1
worker01       Ready    worker                 307d   v1.27.4+k3s1
worker03       Ready    worker                 307d   v1.27.4+k3s1
```

또한, 추가한 라벨도 확인합니다. (`kubectl get nodes --show-labels`)

여러 개의 라벨이 있는게 당연하니, 우리가 추가한 `node-type=worker` 가 잘 있는지만 확인하면 됩니다.

### 마치며

축하합니다! 하드웨어 및 쿠버네티스 설치를 성공적으로 마쳤습니다.

긴 설정인 만큼, 진행이 막히는 부분이 있으면 편하게 댓글로 알려주세요!