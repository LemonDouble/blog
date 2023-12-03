---
title: "내가 보려고 쓰는 K8S 리소스 정리"
description: 
date: 2023-01-26T23:15:42+09:00
image: cover.png
math: 
categories:
    - K8S
tags:
    - K8S
---

* `k get pod <podname> -o` 또는 `k describe pod <podname>`: 상태 정보 자세히 확인
* `k logs -f <podname> -c <container-name>` : 로그 확인 
* `k get pod <podname> --show-lables` : Label 정보 확인

### 1. Pod

* 하나 이상의 Container의 집합
* 실행의 최소 단위

* 특징
  * 여러 Container로 구성되었다면, 항상 동일 Node에 할당
  * 고유 Pod IP 할당 (Pod당 하나)
  * Pod 내의 Container들은 네트워크 공유, 따라서 Localhost로 접근 가능
  * Volume 공유

* Pod이 가질 수 있는 State
  * `Pending` : Master Node에 생성 명령은 전달되었지만 생성되지 않은 상태
  * `ContainerCreating` : 특정 Node에 할당되어 생성 중 (이미지 다운로드 등..)
  * `Running` : 정상 실행 중
  * `Completed` : 한번 실행되고 종료되는 Pod (배치 작업 등..) 작업이 완료된 경우
  * `Error` : Pod에 문제 발생한 경우
  * `CrashLoopBackOff` : 지속적으로 에러가 발생해 Crash가 반복되는 상태

* spec.restartPolicy
  * `Always` : 종료시 항상 재시작 (default)
  * `Never` : 재시작 X
  * `OnFailure` : 실패한 경우에만 재시작 (정상 종료되면 재시작되지 않음)

* 상태 확인
  * `livenessProbe` : Pod이 정상적으로 동작하고 있는지 확인
  * `readnessProbe` : Pod이 생성되고 난 후, 트래픽을 받을 준비가 완료되었는지 확인 (READY 0/1 상태로 확인 가능) 

### 2. Service

* Reverse Proxy 생각하면 쉬움. Pod은 불안정한데 그 앞에 서서 안정적인 Endpoint 제공
* DNS와 비슷하게 Service 이름으로 연결 가능. myservice라는 service를 생성했다면 myservice라는 이름으로 연결 가능 

* Service의 DNS
  * `<ServiceName>.<Namespace>.svc.cluster.local`
  * 예시 : default Namespace의 myservice -> `myservice.default.svc.cluster.local`
  * svc.cluster.local은 K8S 기본값

* Service는 어떻게 DNS를 가질까?
  * `k exec client -- cat /etc/resolve.conf` 로 Pod의 DNS 설정 확인 -> Nameserver의 IP 확인 가능
  * `k get svc -n kube-system` 입력하면 나오는 CoreDNS가 해당 IP
  * CoreDNS라는 서비스 통해 DNS 사용 가능

* Service의 종류
  * `ClusterIP` : (default) K8S 클러스터 내부에서만 접근 가능.
  * `NodePort` : Localhost의 특정 Port를 Service의 특정 포트와 연결 (Docker 생각하기)
    * 단, Nodeport는 모든 Node의 특정 포트를 해당 Service로 연결
    * 예를 들어 32000번 Nodeport가 있다면, `master:32000`, `worker01:30002` 전부 32000번 포트로 연결됨
  * `LoadBalancer` : Load Balancer 생성하여(Cloud 제공, 혹은 Software) Pod에 분산
    * 외부에선 LoadBalancer의 IP만 알면 되니 편하다. (ClusterIP -> Pod의 안정적 서비스 Endpoint, LoadBalancer -> Node의 안정적 서비스 Endpoint)
    * External-IP를 통해서 외부에서 접근 가능
  * `ExternalName` : 외부 서비스를 K8S 네트워크 내에서 사용하고 싶을 때
    * ex : google.com을 ExternalName으로 google-svc 로 설정하면 `curl google-svc` 로 google.com의 데이터를 불러올 수 있다.

### 3. Controller

* Current State와 Desired State를 비교해 일치하도록 Task를 반복

* `ReplicaSet`
  * Pod을 복제해 일정 개수를 유지시킴.
  * spec.replicas : 2 와 같이 복제할 Pod의 개수를 선언 가능
  * selector.matchLables 를 통하여 label이 Match되는 Pod을 선택 가능

* `Deployment`

  * ReplicaSet을 이용하여 일정 개수의 Pod을 생성, 거기에 배포 기능을 추가 (Deployment -> ReplicaSet -> Pod)
  * Rolling Update 지원, 업데이트 되는 Pod의 비율 조절 가능
    * `spec.strategy.type : RollingUpdate` 로 Update 전략 설정
    * `spec.strategy.rollingUpdate.maxUnavailable : 25%` : (소숫점 내림) 배포중 최대 중단 가능 Pod의 % 지정. 10개에 25%라면 2개 Pod
    * `spec.strategy.rollingUpdate.maxSurge : 25%` : (소숫점 올림) 배포중 최대 초과 가능 Pod의 % 지정. 10개에 25%라면 3개 Pod
  * Update History 저장 및 Rollback 가능
    * `k rollout undo deployment <Deployment이름>`

* StatefulSet
  * Pod마다 고유한 식별자 존재
  * Pod마다 고유한 데이터를 스토리지에 보관 가능 (예시 : Database) 
  * Pod생성의 순서 보장 -> 순서에 민감한 어플리케이션 (첫 어플리케이션은 Master, 두번째 이후는 Slave 등) 에 사용
  * `name-<순번>` (0 시작) 식으로 Pod 이름 부여
  * 삭제되는 경우 역순 삭제 (숫자가 높은것부터 삭제)

* DaemonSet
  * 모든 Node에 동일한 Pod을 실행시키고 싶을 때 -> 모니터링, 로그 수집 등

### 4. Job

* `Job`
  * 단발성으로 실행되고 끝나는 프로그램 (ML 학습, Batch Job등)
  * `spec.backoffLimit : 2` 와 같이 실패시 재시도 정책을 정할 수 있음. (2회 재시도)  

* CronJob
  * 주기적으로 Job을 실행할 수 있도록 Crontab과 같은 기능 제공
  * `spec.schedule : "1/* * * * *"` 과 같이 지정

### Namespace

* 클러스터를 논리적으로 나누는 데 사용


* default : 기본 Namespace
* kube-system : K8S의 핵심 컴포넌트 (DNS, 네트워크 등..)의 Pod이 존재
* kube-public : 외부로 공개 가능한 리소스를 담는 Namespace
* kube-node-lease : Node가 살아있는지 master에 알리는 용도? 의 namespace

### ConfigMap, Secret

* `ConfigMap`
  * Metadata (설정값) 저장
  * Key-value 형태로 값 저장

  * 다음과 같이 Yaml로 선언 가능
  ```yaml
  # game-volume.yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: game-volume
  spec:
    restartPolicy: OnFailure
    containers:
    - name: game-volume
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh", "-c", "cat /etc/config/game.properties" ]
      volumeMounts:
      - name: game-volume
        mountPath: /etc/config
    volumes:
    - name: game-volume
      configMap:
        name: game-config
  ```

  * 이후 다른 Pod에서 Volume 연결/환경변수로 사용 가능
    * Volume : 
  ```yaml
  .....
      volumeMounts:
    - name: game-volume
      mountPath: /etc/config
  volumes:
  - name: game-volume
    configMap:
      name: game-config
  ```
  * Env :
  ```yaml
  .....
    envFrom:
    - configMapRef:
        name: monster-config
  ```

* Secrets
  * tmpfs (메모리를 파일처럼 다룰 수 있게 하는 시스템) 사용해 보안에 이점 있음
  * 조회시 Base64 인코딩 (암호화 X, 단지 인코딩 한번 해 줌)

### Appendix 1. 특정 Node에서만 Pod 실행하기

* 예시 : SSD/HDD Node를 구분하여 배치하는 방법

* 각 Node에 label 부착
  * `kubectl label node master disktype=ssd`
  * `kubectl label node worker disktype=hdd`

* 이후 nodeSelector로 label 선택
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-selector
spec:
  containers: 
  - name: nginx
    image: nginx
  # 특정 노드 라벨 선택
  nodeSelector:
    disktype: ssd
```