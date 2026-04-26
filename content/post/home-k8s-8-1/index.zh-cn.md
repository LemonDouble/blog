---
title: "在家用树莓派集群搭建数据中心 - 8-1. Docker Registry UI 配置与 Registry GC 使用方法"
slug: a68cc3e1f5e8a95b
date: 2023-12-10T11:39:46+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - 树莓派
tags:
    - K8S
    - K3S
    - 树莓派
    - Docker Registry
    - GC
    - Web UI
---

_注：作者居住在韩国，部分内容包含韩国特有的背景。_

### 1. Private Registry 虽然好用，但管理起来太麻烦了！

虽然有些人非常喜欢 CLI，但我实在记不住命令，所以更喜欢 GUI 和有迹可循的记录。

Docker Registry 也可以通过 CLI 进行管理，但为了更方便一些，我使用 [docker-registry-browser](https://github.com/klausmeyer/docker-registry-browser)。

将会使用类似下图的 GUI。

![Alt text](image.png)

![Alt text](image-1.png)

### 2. 安装

将其与现有的 docker-registry-system 一起安装。

请在原有文件下方添加 `---` 后再编写。

`modules/docker-registry-system/deployment.yaml`

- 现有 deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: docker-registry-ui-deployment
  namespace: docker-registry-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: docker-registry-ui-app
  template:
    metadata:
      labels:
        app: docker-registry-ui-app
    spec:
      nodeSelector:
        node-type: worker
      containers:
        - name: docker-registry-ui-pod
          image: klausmeyer/docker-registry-browser:1.7.0
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8080
          envFrom:
            - secretRef:
                name: docker-registry-ui-secrets-sealed-secrets
```

`modules/docker-registry-system/service.yaml`

```yaml
... 现有 service
---
apiVersion: v1
kind: Service
metadata:
  name: docker-registry-ui-service
  namespace: docker-registry-system
spec:
  selector:
    app: docker-registry-ui-app
  type: ClusterIP
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
```

`modules/docker-registry-system/ingress.yaml`

```yaml
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: docker-registry-ui-ingress
  namespace: docker-registry-system
spec:
  tls:
    certResolver: le
  routes:
    - kind: Rule
      match: Host(`用于展示 UI 的 URL 示例:docker-ui.lemon.com`)
      services:
        - name: docker-registry-ui-service
          port: 80
```

如果还想配置 Basic Auth，请参考 [通过 Sealed Secrets 管理密钥 + Traefik Basic Auth 设置](/zh-cn/p/1379182f14a62e9b/)。


`modules/docker-registry-system/sealed-docker-registry-ui-secrets.yaml`

进入 sealed secrets GUI，添加以下 Secret。（参考 Docs [Link](https://github.com/klausmeyer/docker-registry-browser/blob/master/docs/README.md)）

- BASIC_AUTH_USER：登录 registry 使用的用户名
- BASIC_AUTH_PASSWORD：密码
- DOCKER_REGISTRY_URL：https://<我注册的 Registry 地址>
- ENABLE_DELETE_IMAGES：true
- SECRET_KEY_BASE：执行 `openssl rand -hex 64` 后得到的值

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: docker-registry-ui-secrets
  namespace: docker-registry-system
  annotations: {}
spec:
  encryptedData:
    BASIC_AUTH_USER: afdfads...
    BASIC_AUTH_PASSWORD: fadvbads..
    DOCKER_REGISTRY_URL: dfd..
    ENABLE_DELETE_IMAGES: afdgd...
    SECRET_KEY_BASE: dffdf...

```

使用方法很直观，建议大家直接动手试一试！


### 3. Registry GC

即使在 Docker Registry UI 中删除镜像（或通过 CLI 删除镜像），磁盘空间也不会立即减少。

这是因为通过 GUI/CLI 删除镜像时，实际并没有发生物理删除，仅删除该镜像的 Manifest。

实际的数据删除发生在显式执行 Registry GC 的时候。

虽然还可以聊得更深入，但今天先聚焦在使用方法上，详细内容就略过。

（如果对相关内容更感兴趣，建议先阅读 [通过透明玻璃纸理论理解 Overlay FS 的使用方法与联合挂载](https://blog.naver.com/alice_k106/221530340759)（韩文）一文，然后再以 Docker registry 与镜像删除等关键词搜索看看。）

* 执行 Registry GC

**!! 注意！GC 执行期间不应在仓库中发生 Pull/Push。请先确认这一点再进行操作！**

1. 使用 `kubectl get pod -n docker-registry-system` 找到 Registry 的 Pod。
2. 执行 `kubectl exec -it pod/<registry-pod-name> -n docker-registry-system -- /bin/registry garbage-collect /etc/docker/registry/config.yml` 来执行 GC。
3. 之后在 Longhorn Dashboard 中选择 Volume -> 点击 Trim Filesystem 来重新计算 Actual Size。

### 4. 结语

通过本文了解了 Docker UI、GC 的方法。

现在已经有了 Private Registry 和管理方法，可以安心地开发（？）服务器，把想要的服务器部署进去！

下次将介绍如何使用 CloudNativePG 在 Kubernetes 上运行数据库。
