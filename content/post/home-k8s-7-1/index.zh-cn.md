---
title: "在家用树莓派集群搭建数据中心 - 7-1. 通过 Web UI 轻松创建 Sealed Secrets"
slug: 567b4c177bdb8401
date: 2023-12-10T00:41:05+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - 树莓派
tags:
    - K8S
    - K3S
    - 树莓派
    - Sealed Secrets
    - Web UI
    - kubeseal
---

_注：作者居住在韩国，部分内容包含韩国特有的背景。_

### 1. 前言

Sealed Secrets 整体都不错，但有一个问题：每次配置时都要创建 Secrets.yaml，再用 CLI 转换为 Sealed Secrets，然后添加到 Git，这个过程实在太繁琐了。

为了让这个过程更轻松，我打算使用 [kubeseal-webgui](https://github.com/Jaydee94/kubeseal-webgui) 简单地搭建一个 Web UI。

![Kubeseal webui，可以在网页上简单设置 Secret 并获取 Sealed Secrets。](image.png)

### 2. 安装

我们再次使用 ArgoCD 来简单安装吧！

`modules/sealed-secrets-system/kubeseal-webgui.yaml`

```yaml
# https://github.com/Jaydee94/kubeseal-webgui
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kubeseal-webgui
  namespace: argocd
spec:
  destination:
    namespace: sealed-secrets-system
    server: 'https://kubernetes.default.svc'
  source:
    repoURL: 'https://jaydee94.github.io/kubeseal-webgui'
    targetRevision: 5.1.4
    chart: kubeseal-webgui
    helm:
      parameters:
        - name: api.url
          value: https://<web ui를 사용할 subdomain 예시:seal.lemon.com>
        - name: sealedSecrets.autoFetchCert
          value: 'true'
        - name: sealedSecrets.controllerName
          value: sealed-secrets
        - name: sealedSecrets.controllerNamespace
          value: sealed-secrets-system
  sources: []
  project: default
```

`modules/sealed-secrets-system/ingress.yaml`

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: sealed-secrets-ingress
  namespace: sealed-secrets-system
spec:
  tls:
    certResolver: le
  routes:
    - kind: Rule
      match: Host(`<web ui를 사용할 subdomain 예시:seal.lemon.com>`)
      services:
        - name: kubeseal-webgui
          port: 8080
```

如有需要，可参考[上一篇文章](https://lemondouble.github.io/zh-cn/p/%EC%A7%91%EC%97%90%EC%84%9C-%EB%9D%BC%EC%A6%88%EB%B2%A0%EB%A6%AC-%ED%8C%8C%EC%9D%B4-%ED%81%B4%EB%9F%AC%EC%8A%A4%ED%84%B0%EB%A1%9C-%EB%8D%B0%EC%9D%B4%ED%84%B0%EC%84%BC%ED%84%B0-%EC%B0%A8%EB%A6%AC%EA%B8%B0-7.-sealed-secrets%EB%A5%BC-%ED%86%B5%ED%95%9C-%EB%B9%84%EB%B0%80-%EA%B4%80%EB%A6%AC--traefik-basic-auth-%EC%84%A4%EC%A0%95/)来添加 Basic Auth。

之后就可以通过该 UI 轻松创建 Sealed Secrets 了！
