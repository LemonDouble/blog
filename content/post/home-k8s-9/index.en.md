---
title: "Setting Up a Home Datacenter with a Raspberry Pi Cluster - 9. Running Databases on Kubernetes with CloudNativePG"
slug: a4ded8cced5f045c
date: 2023-12-12T22:09:10+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - Raspberry Pi
tags:
    - K8S
    - K3S
    - Raspberry Pi
    - CloudNativePG
    - PostgreSQL
    - Operator
    - HA
---

_Note: I'm based in Korea, so some context here is Korea-specific._

### Introduction

Databases are tricky to manage.

Unlike other Pods, you have to worry about data storage, backups, and management, and you also have to pay attention to failover and performance.

That's why I've heard it's common to use a managed system like AWS RDS or a separate instance just for the DB, even if you run other workloads on a Kubernetes cluster.

But does that really matter? If you're setting up a datacenter at home, shouldn't you have at least one managed DBMS?

So let's build one. We'll use [CloudNativePG](https://cloudnative-pg.io/), which leverages Kubernetes' Operator Pattern, to deploy a cluster with one Primary and two Replicas, and configure it to be accessible from the internal network.

It's not a Postgres Operator, but if you read [Running MySQL DB on Kubernetes with the MySQL Operator](https://nangman14.tistory.com/79#5.%20Mysql%20Operator%20Failover%20%ED%85%8C%EC%8A%A4%ED%8A%B8-1), it might help you follow along a bit better.

### 2. Installation

I'll split the installation into two stages.

1. Install the CNPG Operator
2. Install the CNPG Cluster

The Operator's job is to monitor whether the Cluster stays in a healthy state.
The actual database cluster you'll use is installed in step 2.

Let's get started step by step! Once again, we'll deploy quickly using ArgoCD.

**1. Installing the CNPG Operator**

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

Simple, right? After installation, deploy it to install the CNPG Operator.

**2. Installing the CNPG Cluster**

The cluster we're going to build looks like this:

1. Daily Backups to S3 at UTC 00:00 (9 AM KST).
2. Made up of 3 Pods total. Each Pod is distributed across multiple nodes to prevent any unforeseen mishaps (?).
3. The DB is accessible from the internal network at the IP 192.168.0.x.

Let's tackle these one by one!

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

For ease of management, I allowed Superuser access, and 
set the DB capacity to 10GB. (You can expand it later.)

Beyond that, in case of an unexpected disaster (...), I configured backups to be saved on AWS S3 for easy backup, and set the retention to a maximum of 90 days.

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

A simple Daily Backup resource.

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

In my case, I allowed access via the address 192.168.0.206.
Afterward, when querying or managing the database, you can manage it from the internal network by accessing `192.168.0.206`.

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

Grant S3FullAccess on AWS, or access permissions to a specific bucket, then issue an ACCESS KEY and Secret. Then register them using the Sealed Secret you set up previously.

`modules/cnpg-cluster/sealed-superuser-secrets.yaml`

The creation process is a bit complex!

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: superuser-secrets
  namespace: cnpg-cluster
type: kubernetes.io/basic-auth
stringData:
  username: <The ID I'll use, raw without b64 encoding>
  password: <The password I'll use, raw without b64 encoding>
```

First, create a Secret like the above with the name secret.yaml, then 

```sh
cat secret.yaml | kubeseal --controller-namespace=sealed-secrets-system --controller-name=sealed-secrets -oyaml > sealed-superuser-secrets.yaml
```

convert it to a Sealed Secret with the above command, and use the Sealed Secret!

After that, wait for provisioning (it takes some time), and you can log in via 192.168.0.206 with the ID/Password you just configured to use the DB!

**If everything is installed correctly, set a reminder for the next day and please MAKE SURE to verify that the backups are properly saved to that S3 folder!!!**

### 3. Recovery

After a day passes and the backup has been performed normally, **please MAKE SURE to verify that recovery actually works**.
It's too late once you've already lost everything...


I'll share my recovery configuration file.

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

  bootstrap: # added
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

  externalClusters: # added
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

If you add the bootstrap and externalClusters options to a new cluster as shown above, it will automatically recover the data using the existing S3 files when the cluster first starts up.

Once you've connected/verified and recovery is complete, remove the bootstrap and externalClusters options and add the original backup options back in.

A word of caution here:
If the major version of Postgres differs, recovery might not work properly.

For example, if I was using a 1.16 Operator (Postgres 15) and updated the Operator to 1.21 (which uses Postgres 16 by default), the existing cluster would still be on PG 15 unless I manually upgrade it.

In this case, if you try to recover after a failure, since the Operator version is 1.21, PG16 will be provisioned, and since the data stored on the existing S3 is PG15 data, recovery may not work.

For situations like this, you can either downgrade the Operator to the version it was at install time and bring up the same version of PG, then recover -> update, or set the Image to use the same major version.

**If possible, I HIGHLY recommend creating a new cluster, verifying that recovery works properly, and only then proceeding with the next steps!!**

### 4. Cluster Updates

Minor updates happen automatically, but for major version updates, doing the following will usually go without issue. (Tried 15 -> 16)

For online updates, refer to [The Current State of Major PostgreSQL Upgrades with CloudNativePG](https://www.enterprisedb.com/blog/current-state-major-postgresql-upgrades-cloudnativepg-kubernetes),

If you're doing offline updates (where downtime is acceptable):

1. Dump data from the existing cluster with pg_dumpall
2. Create a new cluster
3. Pour the data dumped in step 1 into 2
4. Change applications that pointed to the old cluster to point to the new cluster
5. Test, then delete the old cluster

### 5. Wrapping Up

I've been running my server with CNPG smoothly for about 7-8 months now.

Considering how often I accidentally unplug cables while cleaning(...), it's robust enough to be used as a reliable Postgres DB in normal situations, and even when I accidentally killed power to the entire cluster, I could easily recover the data from the existing S3, so I felt this is a fairly trustworthy system.

Of course, if you have the budget, using a managed RDB would be the best option. But as a proof of technology, I'd appreciate it if you also know that systems like this exist!

At this point, you should have all the systems needed for your server, etc., set up so you can start development. Let's build things one at a time, starting simple.

And within Kubernetes, the DB can be accessed by the name cnpg-lb-rw.cnpg-cluster. For example, like `jdbc:postgresql://cnpg-lb-rw.cnpg-cluster:5432/my_app`.

Thanks for reading this long post! Next time, I'll cover how to use GPUs in K3S with the nvidia-device-plugin!
