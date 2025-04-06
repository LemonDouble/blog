---
title: "집에서 라즈베리 파이 클러스터로 데이터센터 차리기 - 9. CloudNativePG로 쿠버네티스에서 데이터베이스 운용하기"
slug: a4ded8cced5f045c
date: 2023-12-12T22:09:10+09:00
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

### 서론

데이터베이스는 관리하기 까다롭습니다.

다른 Pod과 다르게 데이터 보관, 백업, 관리에도 신경써야 하고, Failover 및 성능에도 관심을 가져야 합니다.

그래서 다른 워크로드는 쿠버네티스 클러스터에서 돌리더라도, DB만큼은 AWS RDS등의 관리형 시스템이나 별도 인스턴스를 사용하는 경우도 흔한 것으로 알고 있습니다.

하지만 그게 중요할까요? 집에서 데이터센터를 차리는데 관리형 DBMS도 하나쯤 있어야 하지 않을까요?

그러니까 한번 만들어 봅시다. 쿠버네티스의 Operation Pattern을 사용한 [CloudNativePG](https://cloudnative-pg.io/) 를 이용해 Primary 하나, Replica 2개짜리 클러스터를 배포해 보고 내부망에서 접속 가능하도록 설정까지 진행해 봅니다.

Postgres Operator는 아니지만 [Mysql Operator로 Kubernetes 환경에서 Mysql DB 운영하기](https://nangman14.tistory.com/79#5.%20Mysql%20Operator%20Failover%20%ED%85%8C%EC%8A%A4%ED%8A%B8-1) 글을 읽어보시면 따라오는데 좀 더 도움이 될?수도 있습니다.

### 2. 설치

설치는 두 단계로 나눠서 진행하겠습니다.

1. CNPG Operator 설치
2. CNPG Cluster 설치

Operator는 Cluster가 정상 상태를 유지하는지 감시하는 역할을 합니다.
실제로 사용하는 데이터베이스 클러스터는 2.를 통해 설치합니다.

차근차근 시작해 봅시다! 이번에도 ArgoCD를 이용해 빠르게 배포합니다.

**1. CNPG Operator 설치**

`apps/enabled/cnpg-system.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cnpg-system
  namespace: argocd
spec:
  destination:
    namespace: cnpg-system
    server: 'https://kubernetes.default.svc'
  source:
    path: modules/cnpg-system
    repoURL: 'git@github.com:<YourOrganizationName>/<YourRepositoryName>.git'
    targetRevision: HEAD
  project: default
```

`modules/cnpg-system/cnpg.yaml`
```yaml
# https://github.com/cloudnative-pg/cloudnative-pg
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cnpg
  namespace: argocd
spec:
  destination:
    namespace: cnpg-system
    server: 'https://kubernetes.default.svc'
  source:
    repoURL: 'https://cloudnative-pg.github.io/charts'
    targetRevision: 0.19.1
    chart: cloudnative-pg
  project: default
```

간단하죠? 설치 이후 배포를 진행해 CNPG Operator를 설치해 줍시다.

**2. CNPG Cluster 설치**

저희가 만들 클러스터의 모습은 다음과 같습니다.

1. 매일 UTC 0시 (KST 기준 아침 9시)에 S3에 Daily Backup을 진행합니다.
2. 총 3개의 Pod으로 구성됩니다. 각 Pod은 여러 노드에 분산되어 혹시 모를 불행을(?) 예방합니다.
3. 192.168.0.x IP로 내부망에서 DB 접근이 가능합니다.

하나하나 시작해 봅시다!

`apps/enabled/cnpg-cluster.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cnpg-cluster
  namespace: argocd
spec:
  destination:
    namespace: cnpg-cluster
    server: 'https://kubernetes.default.svc'
  source:
    path: modules/cnpg-cluster-16
    repoURL: 'git@github.com:<YourOrganizationName>/<YourRepositoryName>.git'
    targetRevision: HEAD
  project: default
```

`modules/cnpg-cluster/cluster.yaml`
```yaml
# https://cloudnative-pg.io/documentation/1.21/quickstart/
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  namespace: cnpg-cluster
  name: cnpg-cluster
spec:
  instances: 3

  superuserSecret:
    name: superuser-secrets
  enableSuperuserAccess: true
  primaryUpdateStrategy: unsupervised

  # Persistent storage configuration
  storage:
    size: 10Gi
    pvcTemplate:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      storageClassName: longhorn-ssd
      volumeMode: Filesystem

  # Backup properties
  backup:
    retentionPolicy: "90d"
    barmanObjectStore:
      destinationPath: s3://lemon-backup/cnpg-backup
      s3Credentials:
        accessKeyId:
          name: aws-backup-secret
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: aws-backup-secret
          key: ACCESS_SECRET_KEY
      wal:
        compression: gzip
```

관리 편의를 위해 Superuser 접근을 허용해 주고, 
DB 용량은 10GB로 주었습니다. (이후 확장 가능합니다.)

이외에 만약 불의의 사고가 닥쳤을 때(...) 쉽게 백업할 수 있도록 백업 스토리지를 AWS S3에 백업 파일을 저장하게 설정하고, 최대 90일까지 보관하도록 하였습니다.

`modules/cnpg-cluster/daily-backup.yaml`

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  namespace: cnpg-cluster
  name: daily-backup
spec:
  schedule: "0 0 0 * * *" # Daily
  backupOwnerReference: self
  cluster:
    name: cnpg-cluster
```

간단한 Daily Backup 리소스입니다.

`modules/cnpg-cluster/lb.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cnpg-lb-rw
  namespace: cnpg-cluster
spec:
  ports:
    - name: postgres
      port: 5432
      protocol: TCP
      targetPort: 5432
  selector:
    cnpg.io/cluster: cnpg-cluster
    role: primary
  type: LoadBalancer
  loadBalancerIP: 192.168.0.206
```

저의 경우 192.168.0.206 주소로 접근을 허용해 주었습니다.
이후 데이터베이스 조회나 관리 시에 내부 망에서 `192.168.0.206` 로 접근하여 관리가 가능해집니다.

`modules/cnpg-cluster/sealed-aws-secrets.yaml`

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: aws-secrets
  namespace: cnpg-cluster
  annotations: {}
spec:
  encryptedData:
    ACCESS_KEY_ID: adffd...
    ACCESS_SECRET_KEY: Aadfads...
```

AWS에서 S3FullAccess, 또는 특정 버킷의 액세스 권한을 준 후 ACCESS KEY, Secret을 발급받은 후, 이전에 설정했던 Sealed Secret을 이용해 등록해 줍니다.

`modules/cnpg-cluster/sealed-superuser-secrets.yaml`

생성 과정이 조금 복잡합니다!

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: superuser-secrets
  namespace: cnpg-cluster
type: kubernetes.io/basic-auth
stringData:
  username: <내가 쓸 ID, b64인코딩 없이 원본 그대로>
  password: <내가 쓸 패스워드, b64인코딩 없이 원본 그대로>
```

위와 같은 Secret을 먼저 secret.yaml이란 이름으로 생성 후, 

```sh
cat secret.yaml | kubeseal --controller-namespace=sealed-secrets-system --controller-name=sealed-secrets -oyaml > sealed-superuser-secrets.yaml
```

로 Sealed Secret으로 변경 후, Sealed Secret을 사용합니다!

이후 프로비저닝을 기다리고, (시간이 좀 걸립니다.) 192.168.0.206으로 방금 설정한 ID/Password를 통해 로그인하여 DB를 사용할 수 있습니다!

**정상적으로 설치되었다면, 다음날 알림을 걸어 꼭!! S3 해당 폴더에 정상으로 백업되는지 확인해 주세요!!!**

### 3.복구

하루가 지나 정상 백업이 되었다면, **꼭! 복원이 되는지 확인** 하시길 바랍니다.
날아가고 확인하면 너무 늦어요...


제 복원 설정 파일을 공유 드립니다.

```yaml
# https://cloudnative-pg.io/documentation/1.17/quickstart/
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  namespace: cnpg-cluster
  name: cnpg-cluster
spec:
  instances: 3

  superuserSecret:
    name: superuser-secrets
  primaryUpdateStrategy: unsupervised

  bootstrap: #추가
    recovery:
      source: clusterBackup

  storage:
    size: 10Gi
    pvcTemplate:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      storageClassName: longhorn-ssd
      volumeMode: Filesystem

  externalClusters: #추가
    - name: clusterBackup
      barmanObjectStore:
        serverName: cnpg-cluster
        destinationPath: s3://lemon-backup/postgres-backup
        s3Credentials:
          accessKeyId:
            name: aws-secrets
            key: ACCESS_KEY_ID
          secretAccessKey:
            name: aws-secrets
            key: ACCESS_SECRET_KEY
        wal:
          compression: gzip
```

신규 클러스터를 위와 같이 bootstrap, externalClusters 옵션을 추가하면 클러스터가 처음 시작될 때, 기존 S3 파일을 이용해 자동으로 데이터를 복구합니다.

접속/ 확인 후 복구가 끝났으면 bootstrap, externalClusters 옵션을 지운 후 기존의 백업 옵션을 다시 추가해 주시면 됩니다.

이때 주의할 점으로,
Postgres의 메이저 버전이 다른 경우 복구가 정상적으로 되지 않을 수 있습니다.

예를 들어 제가 1.16버전 Operator를 사용하다 (Postgres 15), Operator를 1.21(기본값으로 Postgres 16 Operator)까지 업데이트를 했다 하더라도 기생성된 클러스터는 수동 업그레이드를 해 주지 않는 이상 PG 15를 유지합니다.

이 떄, 만약 장애가 생겨 복구를 시도하는 경우 Operator 버전이 1.21 버전이므로 PG16이 프로비저닝 될 것이고, 이 경우 기존 S3에 저장된 데이터가 PG15 데이터이므로 복구가 안 될 수 있습니다.

이런 경우는 Operator를 설치 당시로 다운그레이드하여 동일 버전의 PG를 올린 후 복구 -> 업데이트를 진행하거나, Image를 설정하여 동일 메이저 버전을 사용하는 법이 있습니다.

**가능하다면, 신규 클러스터를 하나 만들어 정상 복구가 되는지 꼭 확인하고, 이후 다음 내용을 진행하는걸 꼭! 추천드립니다!!** 

### 4. 클러스터 업데이트

마이너 업데이트의 경우는 자동으로 진행되나, 메이저 버전 업데이트시에는 다음과 같이 진행하면 보?통 별문제 없을겁니다. (15 -> 16 진행해 봄)

온라인 업데이트의 경우는 [The Current State of Major PostgreSQL Upgrades with CloudNativePG](https://www.enterprisedb.com/blog/current-state-major-postgresql-upgrades-cloudnativepg-kubernetes) 를 참고하시고,

오프라인 업데이트 (다운타임이 발생해도 문제X) 라면

1. 기존 클러스터에서 pg_dumpall로 데이터 덤프
2. 신규 클러스터 생성
3. 1에서 덤프뜬 데이터를 2에 부음
4. 기존 클러스터를 바라보던 어플리케이션을 신규 클러스터를 바라보도록 변경
5. 테스트 후 기존 클러스터 삭제

### 5. 미치며

저는 현재 CNPG를 이용해 약 7~8개월간 서버를 정상적으로 운영하고 있습니다.

제가 청소하다 선을 자주 빼먹는걸 감안하면(...) 일반적인 상황에서 충분히 견고한 Postgres DB로 사용될 수 있고, 제가 실수로 클러스터 전원을 전부 날렸을 때도 기존 S3에서 간단히 데이터를 복구할 수 있어 신뢰성이 꽤 높은 시스템이라는 생각을 하였습니다.

물론, 돈의 여유가 있으면 관리형 RDB를 쓰는게 가장 좋은 선택이겠습니다만, 기술증명 겸 이러한 시스템이 있다는 것도 알아주셨으면 감사하겠습니다!

이 시점에서, 서버 등에 필요한 시스템이 모두 갖춰져 개발을 시작할 수 있게 되셨을 겁니다. 간단한 것부터 하나씩 만들어 봐요.

그리고 쿠버네티스 내에서 해당 DB는 cnpg-lb-rw.cnpg-cluster 라는 이름으로 접근할 수 있습니다. 예를 들면 `jdbc:postgresql://cnpg-lb-rw.cnpg-cluster:5432/my_app` 처럼요.

긴 글 읽느라 수고 많으셨습니다! 다음엔 nvidia-device-plugin을 이용하여 K3S에서 GPU 사용 법을 알아보겠습니다!