---
title: "自宅でラズベリーパイクラスタによるデータセンター構築 - 7-1. Web UIを使って簡単にSealed Secretsを作成する"
slug: 567b4c177bdb8401
date: 2023-12-10T00:41:05+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - ラズベリーパイ
tags:
    - K8S
    - K3S
    - ラズベリーパイ
    - Sealed Secrets
    - Web UI
    - kubeseal
---

_注：筆者は韓国在住のため、本文には韓国特有の文脈が含まれることがあります。_

### 1. はじめに

Sealed Secretsは全体的に良いのですが、一つ問題があります。毎回設定するたびにSecrets.yamlを作成し、CLIでSealed Secretsに変換してから、Gitに追加するのが非常に面倒だという点です。

これをもう少し楽にするため、[kubeseal-webgui](https://github.com/Jaydee94/kubeseal-webgui)を使って簡単にWeb UIを作ってみる予定です。

![Kubeseal webui、簡単にウェブからSecretを設定してSealed Secretsを受け取れる。](image.png)

### 2. インストール

ArgoCDを使ってまた簡単にインストールしてみましょう！

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

必要であればBasic Authは[前回の記事](https://lemondouble.github.io/ja/p/%EC%A7%91%EC%97%90%EC%84%9C-%EB%9D%BC%EC%A6%88%EB%B2%A0%EB%A6%AC-%ED%8C%8C%EC%9D%B4-%ED%81%B4%EB%9F%AC%EC%8A%A4%ED%84%B0%EB%A1%9C-%EB%8D%B0%EC%9D%B4%ED%84%B0%EC%84%BC%ED%84%B0-%EC%B0%A8%EB%A6%AC%EA%B8%B0-7.-sealed-secrets%EB%A5%BC-%ED%86%B5%ED%95%9C-%EB%B9%84%EB%B0%80-%EA%B4%80%EB%A6%AC--traefik-basic-auth-%EC%84%A4%EC%A0%95/)を参考に追加してください。

その後、このUIを通じて簡単にSealed Secretsを作成できます！
