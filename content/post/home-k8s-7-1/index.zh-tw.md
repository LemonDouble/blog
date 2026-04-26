---
title: "在家用樹莓派叢集架設資料中心 - 7-1. 透過 Web UI 輕鬆建立 Sealed Secrets"
slug: 567b4c177bdb8401
date: 2023-12-10T00:41:05+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - 樹莓派
tags:
    - K8S
    - K3S
    - 樹莓派
    - Sealed Secrets
    - Web UI
    - kubeseal
---

_註：筆者居住於韓國，部分內容包含韓國特有的背景。_

### 1. 前言

Sealed Secrets 整體上都不錯，但有一個問題：每次設定時都要建立 Secrets.yaml，再用 CLI 轉換為 Sealed Secrets，然後加入 Git，這個過程實在非常繁瑣。

為了讓這個過程更輕鬆，我打算使用 [kubeseal-webgui](https://github.com/Jaydee94/kubeseal-webgui) 簡單地建立一個 Web UI。

![Kubeseal webui，可以在網頁上簡單設定 Secret 並取得 Sealed Secrets。](image.png)

### 2. 安裝

我們再次使用 ArgoCD 來簡單安裝吧！

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

如有需要，可參考[上一篇文章](https://lemondouble.github.io/zh-tw/p/%EC%A7%91%EC%97%90%EC%84%9C-%EB%9D%BC%EC%A6%88%EB%B2%A0%EB%A6%AC-%ED%8C%8C%EC%9D%B4-%ED%81%B4%EB%9F%AC%EC%8A%A4%ED%84%B0%EB%A1%9C-%EB%8D%B0%EC%9D%B4%ED%84%B0%EC%84%BC%ED%84%B0-%EC%B0%A8%EB%A6%AC%EA%B8%B0-7.-sealed-secrets%EB%A5%BC-%ED%86%B5%ED%95%9C-%EB%B9%84%EB%B0%80-%EA%B4%80%EB%A6%AC--traefik-basic-auth-%EC%84%A4%EC%A0%95/)來新增 Basic Auth。

之後就可以透過該 UI 輕鬆建立 Sealed Secrets 了！
