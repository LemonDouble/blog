---
title: "在家用树莓派集群搭建数据中心 - 4. Traefik HTTPS 认证及 ArgoCD 配置"
slug: f10efbc0bcc4c537
date: 2023-12-06T05:21:00+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - 树莓派
tags:
    - K8S
    - K3S
    - 树莓派
    - Traefik
    - Let's Encrypt
    - HTTPS
    - ArgoCD
    - GitOps
---

_注：作者居住在韩国，部分内容包含韩国特有的背景。_

如果通过 Git 来管理整个系统会怎么样？
将所需的容器种类、数量等全部声明在 Git 中，集群完全按照 Git 中声明的内容运行，这样不是非常方便吗？

不必特地查看集群的各个角落，只需查看上传到 Git 的配置值即可。

而且，如果在部署过程中出现问题，不就可以立即 Revert 吗？另外，多人管理同一集群时产生的问题也可以通过我们经常使用的 PR 来管理。

最后，如果集群突然出现问题导致所有节点宕机，由于声明的状态值在 Git 中，即使通过其他集群也能快速进行恢复。

这种方法被称为 GitOps，而 ArgoCD 是帮助将 Git 中声明的状态与集群状态保持一致的工具。

接下来需要部署的系统真的非常多，所以让我们先安装帮助实现 GitOps 的 ArgoCD。

同时为了使 ArgoCD 可以从外部访问，让我们一起进行 Traefik 的 LetsEncrypt 配置等工作。

### 应用 Traefik Letsencrypt Certification

- 参考文章：[HTTPS on Kubernetes Using Traefik Proxy](https://traefik.io/blog/https-on-kubernetes-using-traefik-proxy/)

在 Control Node 上参考以下内容创建一个 traefik-update.yaml。

```yaml
# https://traefik.io/blog/https-on-kubernetes-using-traefik-proxy/
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    ports:
      web:
        redirectTo:
          port: websecure
          priority: 10
    additionalArguments:
      - "--log.level=INFO"
      - "--certificatesresolvers.le.acme.email=yourMail@gmail.com"
      - "--certificatesresolvers.le.acme.storage=/data/acme.json"
      - "--certificatesresolvers.le.acme.tlschallenge=true"
      - "--certificatesresolvers.le.acme.caServer=https://acme-v02.api.letsencrypt.org/directory"
```

然后使用 `kubectl apply -f traefik-update.yaml` 应用到集群。

之后，在 IngressRoute 中可以使用 le（letsEncrypt）作为 CertificationResolver。（如果不明白这是什么意思，继续操作下去就会理解。）

### ArgoCD 安装

让我们安装 ArgoCD，并接入 Github SSO。

1. 进行 DNS 配置。将从外部访问 ArgoCD 时使用的子域名与当前 IP 地址关联。
    - 访问"查看我的 IP"服务确认自己的 IP。
    - 进入购买 DNS 的网站，在 A 记录的 IP4 Address 中填入刚才确认的自己的 IP。

2. 创建一个 `Github Organization`，并创建合适的 Role。我的情况下，是在个人 Organization 中添加了名为 Admin 的 Role。

3. 进入 Organization -> Settings -> Developer Settings -> OAuth Apps，添加一个新的 OAuth 应用。

![ArgoCD OAuth 配置，Authorization callback URL 很重要](image.png)

此时，将 Authorization callback URL 设置为 `https://<刚才设置的域名>/api/dex/callback`。
其余可以自由更改。

之后发放并记住 `Client ID, Secret`。

4. 创建 values.yaml 文件

参考以下文件编写 `values.yaml` 文件。

```yaml
configs:
  params:
    # 由于 Traefik 处理 Https 连接，因此以 Insecure 模式启动
    server.insecure: true
  rbac:
    # Built-in Policy : https://github.com/argoproj/argo-cd/blob/master/assets/builtin-policy.csv
    policy.default: role:readonly
    policy.csv: |
      g, <Github Organization 名称>:<创建的 Role 名称>, role:admin

server:
  config:
    url: https://<设置的域名>

    dex.config: |
      connectors:
      - type: github
        id: github
        name: github
        config:
          clientID: <刚才发放的 Client ID>
          clientSecret: <刚才发放的 Client Secret>
          orgs:
          - name: <Github Organization 名称>
```

例如：

```yaml
configs:
  params:
    # 由于 Traefik 处理 Https 连接，因此以 Insecure 模式启动
    server.insecure: true
  rbac:
    # Built-in Policy : https://github.com/argoproj/argo-cd/blob/master/assets/builtin-policy.csv
    policy.default: role:readonly
    policy.csv: |
      g, LemonLemon:Admin, role:admin

server:
  config:
    url: https://argo.lemon.com

    dex.config: |
      connectors:
      - type: github
        id: github
        name: github
        config:
          clientID: vadf45a6sdfad
          clientSecret: adsfvasd45f6ads4f56asd7
          orgs:
          - name: LemonLemon
```

5. 创建 ingress.yaml 文件

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: argocd-ingress
  namespace: argocd
spec:
  entryPoints:
    - websecure
  routes:
    - kind: Rule
      match: Host(`<设置的域名>`)
      priority: 10
      services:
        - name: argo-cd-argocd-server
          port: 80
    - kind: Rule
      match: Host(`<设置的域名>`) && Headers(`Content-Type`, `application/grpc`)
      priority: 11
      services:
        - name: argo-cd-argocd-server
          port: 80
          scheme: h2c
  tls:
    certResolver: le
```

以下是示例。

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: argocd-ingress
  namespace: argocd
spec:
  entryPoints:
    - websecure
  routes:
    - kind: Rule
      match: Host(`argo.lemon.com`)
      priority: 10
      services:
        - name: argo-cd-argocd-server
          port: 80
    - kind: Rule
      match: Host(`argo.lemon.com`) && Headers(`Content-Type`, `application/grpc`)
      priority: 11
      services:
        - name: argo-cd-argocd-server
          port: 80
          scheme: h2c
  tls:
    certResolver: le
```

6. 编写完 `values.yaml` 和 `ingress.yaml` 后，输入以下命令安装 argocd。

```sh
kubectl create namespace argocd
helm repo add argo https://argoproj.github.io/argo-helm
helm install -n argocd argo-cd argo/argo-cd -f values.yaml

kubectl apply -f ingress.yaml
```

稍等片刻后，访问设置的域名（我的情况是 `https://argo.lemon.com`），ArgoCD 会正常出现。

之后点击 `Log in Via Github` 按钮登录。

如果登录成功，则表示安装成功！

※ 提示：如果出现 401 错误，请尝试 kill 一次 argocd 命名空间的 argocd-server pod。

### ArgoCD Git Private Repo 连接

如果集群使用 Public Repo，可以省略此设置，但为了防止万一发生事故，让我们在 Git 中也使用 private Repo，并在 ArgoCD 中连接使用。

在 Github 中添加一个新的（空的）仓库后，将该仓库与 ArgoCD 关联。

1. 登录 ArgoCD 后，点击 Settings -> Repository -> Connect Repo。
2. connection method 选择 ssh，Name 任意，Project 为 default，Repository URL 填入 git clone 时的 `ssh 地址`，SSH private key data 复制粘贴 Control01 节点的 `~/.ssh/id_rsa`。
3. 连接 Git。

![Github 关联示例](image-1.png)

### 结语

今天我们完成了 Traefik Ingress 配置、ArgoCD 安装和 Git 关联。

下一篇文章将整理至今散落各处(?)的 yaml 文件，使用 App of Apps 整理仓库结构，并构建一个即使发生故障也能通过一次点击恢复的系统。

一如既往，如有错误请告知！
