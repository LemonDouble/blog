---
title: "K8S Resources Cheat Sheet (For My Own Reference)"
slug: f13e7c85c56b2c7c
date: 2023-01-26T23:15:42+09:00
image: cover.png
math: 
categories:
    - K8S
tags:
    - K8S
    - Pod
    - Service
    - Deployment
    - Cheat Sheet
---

_Note: I'm based in Korea, so some context here is Korea-specific._

* `k get pod <podname> -o` or `k describe pod <podname>`: Check detailed status info
* `k logs -f <podname> -c <container-name>`: Check logs
* `k get pod <podname> --show-lables`: Check label info

### 1. Pod

* A group of one or more Containers
* The smallest unit of execution

* Characteristics
  * If composed of multiple Containers, they are always assigned to the same Node
  * Each Pod gets a unique Pod IP (one per Pod)
  * Containers inside a Pod share the network, so they can be reached via Localhost
  * Volumes are shared

* States a Pod can have
  * `Pending`: The creation command was sent to the Master Node but the Pod hasn't been created yet
  * `ContainerCreating`: Assigned to a specific Node and currently being created (image download, etc.)
  * `Running`: Running normally
  * `Completed`: A Pod that runs once and terminates (batch jobs, etc.) after the work is done
  * `Error`: Something went wrong with the Pod
  * `CrashLoopBackOff`: A state where errors keep happening and Crashes repeat continuously

* spec.restartPolicy
  * `Always`: Always restart on termination (default)
  * `Never`: Never restart
  * `OnFailure`: Restart only on failure (no restart on normal termination)

* Health Checks
  * `livenessProbe`: Checks whether the Pod is operating normally
  * `readinessProbe`: Checks whether the Pod is ready to accept traffic after creation (visible as the READY 0/1 status)

### 2. Service

* Easy to think of as a Reverse Proxy. Pods are unstable, so a Service stands in front and provides a stable Endpoint
* Similar to DNS, you can connect by Service name. If you create a service called myservice, you can connect to it by the name myservice

* Service DNS
  * `<ServiceName>.<Namespace>.svc.cluster.local`
  * Example: myservice in the default Namespace -> `myservice.default.svc.cluster.local`
  * svc.cluster.local is the K8S default

* How does a Service get a DNS?
  * Run `k exec client -- cat /etc/resolv.conf` to check the Pod's DNS settings -> you can find the Nameserver IP
  * Running `k get svc -n kube-system` shows CoreDNS, which has that IP
  * DNS is available through the CoreDNS service

* Types of Services
  * `ClusterIP`: (default) Accessible only from within the K8S cluster.
  * `NodePort`: Connects a specific port on Localhost to a specific port on the Service (think Docker)
    * However, NodePort connects a specific port on every Node to the Service
    * For example, if there's a NodePort 32000, then `master:32000`, `worker01:32000` all connect to port 32000
  * `LoadBalancer`: Creates a Load Balancer (provided by the Cloud or Software) and distributes traffic to Pods
    * Convenient since external clients only need to know the LoadBalancer's IP. (ClusterIP -> stable service Endpoint for Pods, LoadBalancer -> stable service Endpoint for Nodes)
    * Accessible externally via External-IP
  * `ExternalName`: When you want to use an external service from within the K8S network
    * e.g., if you set google.com as an ExternalName called google-svc, you can fetch google.com's data with `curl google-svc`

### 3. Controller

* Compares Current State and Desired State and repeatedly performs tasks to make them match

* `ReplicaSet`
  * Replicates Pods to maintain a certain count.
  * You can declare the number of replicas with something like spec.replicas: 2
  * You can select Pods whose labels match via selector.matchLables

* `Deployment`

  * Uses ReplicaSet to create a certain number of Pods, with deployment features added on top (Deployment -> ReplicaSet -> Pod)
  * Supports Rolling Update, with adjustable update ratios
    * Set the update strategy with `spec.strategy.type: RollingUpdate`
    * `spec.strategy.rollingUpdate.maxUnavailable: 25%`: (rounded down) Specifies the max % of Pods that can be unavailable during deployment. With 10 Pods at 25%, that's 2 Pods
    * `spec.strategy.rollingUpdate.maxSurge: 25%`: (rounded up) Specifies the max % of Pods that can exceed the desired count during deployment. With 10 Pods at 25%, that's 3 Pods
  * Stores update history and supports Rollback
    * `k rollout undo deployment <deployment-name>`

* StatefulSet
  * Each Pod has a unique identifier
  * Each Pod can store unique data in storage (example: Database)
  * Pod creation order is guaranteed -> used for order-sensitive applications (first one is Master, the rest are Slaves, etc.)
  * Pods are named like `name-<index>` (starting from 0)
  * On deletion, Pods are deleted in reverse order (highest number first)

* DaemonSet
  * When you want to run the same Pod on every Node -> monitoring, log collection, etc.

### 4. Job

* `Job`
  * One-shot programs that run and finish (ML training, batch jobs, etc.)
  * You can set a retry policy on failure with something like `spec.backoffLimit: 2`. (2 retries)

* CronJob
  * Provides Crontab-like functionality so you can run Jobs periodically
  * Specified like `spec.schedule: "*/1 * * * *"`

### Namespace

* Used to logically partition the cluster


* default: The default Namespace
* kube-system: Holds the Pods of K8S core components (DNS, networking, etc.)
* kube-public: A Namespace that holds resources that can be exposed externally
* kube-node-lease: A Namespace used (I think?) for Nodes to signal to the master that they're alive

### ConfigMap, Secret

* `ConfigMap`
  * Stores Metadata (configuration values)
  * Stores values in Key-Value form

  * Can be declared in YAML like the following
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

  * Other Pods can then use it via Volume mount or environment variables
    * Volume: 
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
  * Env:
  ```yaml
  .....
    envFrom:
    - configMapRef:
        name: monster-config
  ```

* Secrets
  * Uses tmpfs (a system that lets you treat memory like a file) for security benefits
  * When you look up the value, it comes back as Base64-encoded (not encrypted, just encoded once)

### Appendix 1. Running Pods Only on Specific Nodes

* Example: How to separate placement between SSD/HDD Nodes

* Attach a label to each Node
  * `kubectl label node master disktype=ssd`
  * `kubectl label node worker disktype=hdd`

* Then select the label with nodeSelector
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-selector
spec:
  containers: 
  - name: nginx
    image: nginx
  # Select a specific node label
  nodeSelector:
    disktype: ssd
```
