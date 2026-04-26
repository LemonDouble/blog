---
title: "在家用树莓派集群搭建数据中心 - 10-1. 使用 Basic Auth 保护内部服务"
slug: 771435ba12d3f38c
date: 2024-01-07T12:41:24+09:00
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
    - Basic Auth
    - Sealed Secrets
    - 认证
---

_注：作者居住在韩国，部分内容包含韩国特有的背景。_

**以下内容与 Sealed Secrets [第 3 节](https://lemondouble.github.io/zh-cn/p/%EC%A7%91%EC%97%90%EC%84%9C-%EB%9D%BC%EC%A6%88%EB%B2%A0%EB%A6%AC-%ED%8C%8C%EC%9D%B4-%ED%81%B4%EB%9F%AC%EC%8A%A4%ED%84%B0%EB%A1%9C-%EB%8D%B0%EC%9D%B4%ED%84%B0%EC%84%BC%ED%84%B0-%EC%B0%A8%EB%A6%AC%EA%B8%B0-7.-sealed-secrets%EB%A5%BC-%ED%86%B5%ED%95%9C-%EB%B9%84%EB%B0%80-%EA%B4%80%EB%A6%AC--traefik-basic-auth-%EC%84%A4%EC%A0%95/#3-sealed-secrets-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC-basic-auth-%EB%93%B1%EB%A1%9D%ED%95%B4%EB%B3%B4%EA%B8%B0) 有较多重叠。如果你已经完成了那一节，并且对 Basic Auth 较为熟悉，可以直接跳过。**

### 1. 前言

使用 Kubernetes 时，会陆续部署各种各样的服务。

当然，仅在内网中提供服务在安全方面是最理想的，但集群管理并不一定总能从内网进行。比如外出时突然出现问题之类的情况也是有可能的。

这种情况下大致有两种解决方案。

1. 使用 VPN，从其他电脑接入内网，使用相关服务
2. 将服务暴露在互联网上，但增加认证机制

我个人选择了第 2 种，并自己搭建了一个 SSO 服务器使用。

不过，由于 SSO 系统的搭建过程相当庞大，这里先从最简单的认证机制开始。

### 2. Basic Auth 的流程

![Basic Auth，来源：MDN](image.png)

Basic Auth 非常简单！

1. 客户端向服务器请求页面。
2. 服务器收到请求，如果没有认证信息，则返回 401 响应并要求认证方式（Basic Auth）。
3. 客户端收到响应后，进行 Basic Auth。
    - 此时，HTTP Authorization Header 中会带上 `Basic ${认证 token}` 的值。
    - 此处的认证 token 是 Base64Encode("userId:password")。
    - 例如 id 为 lemondouble，密码为 1q2w3e4r! 时，使用对 `"lemondouble:1q2w3e4r!"` 进行 base64 编码后的值 `bGVtb25kb3VibGU6MXEydzNlNHIh`。
4. 服务器验证该 token 后，返回正常响应。

### 3. Basic Auth 的优缺点

优点：

1. 内置！ 大多数浏览器无需额外设置即可支持。
2. 实现简单。 大多数 Web 框架都默认支持！

缺点：

1. 安全性较弱：这是最大的缺点。
   - 密码以 base64 编码（=明文）的形式在每次请求中传输。
     - 如果使用 HTTPS 协议还好，但若使用 HTTP 协议，密码会在每次请求中以明文形式发送！
   - 不防御暴力破解等攻击。
     - 这虽然也算是 Id/Password 本身的问题，但在 Traefik 的默认配置下，并没有诸如丢弃失败请求超过一定次数的客户端之类的功能。
     - 如果 Id/Password 较弱，恶意攻击者可能通过暴力破解访问该服务。

因此，如有可能，建议在以下情形下使用该功能：

1. 即使被访问也无大碍的服务，或仅希望分享给少数用户的服务
2. 使用 HTTPS 协议
3. 使用足够长的密码

### 4. Basic Auth 的配置方法

- 我们将使用 Sealed Secrets 对接入信息（Id/Password）进行加密，并保存到 Git 中。
- 这样做的原因如下：
  - 在突发故障时，连同接入信息一起快速恢复集群（Secrets 信息也在 Git 中，只需部署即可）
  - 即便使用 Private Repo，也可以防止因一时疏忽导致 Id/Password 泄露

Disclaimer：以下内容与 Sealed Secrets [第 3 节](https://lemondouble.github.io/zh-cn/p/%EC%A7%91%EC%97%90%EC%84%9C-%EB%9D%BC%EC%A6%88%EB%B2%A0%EB%A6%AC-%ED%8C%8C%EC%9D%B4-%ED%81%B4%EB%9F%AC%EC%8A%A4%ED%84%B0%EB%A1%9C-%EB%8D%B0%EC%9D%B4%ED%84%B0%EC%84%BC%ED%84%B0-%EC%B0%A8%EB%A6%AC%EA%B8%B0-7.-sealed-secrets%EB%A5%BC-%ED%86%B5%ED%95%9C-%EB%B9%84%EB%B0%80-%EA%B4%80%EB%A6%AC--traefik-basic-auth-%EC%84%A4%EC%A0%95/#3-sealed-secrets-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC-basic-auth-%EB%93%B1%EB%A1%9D%ED%95%B4%EB%B3%B4%EA%B8%B0) 完全相同！

让我们将 Longhorn Dashboard 配置为可从外部访问，同时学习 Basic Auth 的配置方法。

1. 执行 `apt install apache2-utils`，以便使用 htpasswd 命令。
2. 使用 `htpasswd -nb <id> <password> | openssl base64`，获取包含 id、password 的 Secret String。
3. 用任意文本编辑器创建 secret.yaml，并参考下面的内容添加 Secret。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: longhorn-system-basic-auth
  namespace: longhorn-system
data:
  users: <步骤 2 中得到的 String>
```

4. 输入 `cat secret.yaml | kubeseal --controller-namespace=sealed-secrets-system --controller-name=sealed-secrets -oyaml > sealed-secrets.yaml`，得到 sealed-secrets.yaml。
5. 参考下面的内容注册 ingress。

`modules/longhorn-system/ingress.yaml`

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: longhorn-dashboard
  namespace: longhorn-system
spec:
  tls:
    certResolver: le
  routes:
    - kind: Rule
      match: Host(`<期望的 subdomain，例如 longhorn.lemon.com>`)
      middlewares:
        - name: basic-auth
          namespace: longhorn-system
      services:
        - name: longhorn-frontend
          port: 80
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: basic-auth
  namespace: longhorn-system
spec:
  basicAuth:
    secret: longhorn-system-basic-auth
```

在 `modules/longhorn-system/sealed-basic-auth-secret.yaml` 中，将刚刚生成的 sealed-secrets.yaml 复制粘贴进去。

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: longhorn-system-basic-auth
  namespace: longhorn-system
spec:
  encryptedData:
    users: adfasdflavasdlfj...
  template:
    metadata:
      creationTimestamp: null
      name: longhorn-system-basic-auth
      namespace: longhorn-system
```

6. 之后进行部署。
7. 访问时会正常弹出 id/password 确认窗口，未登录则无法看到该页面！

### 5. 结语

Basic Auth 是我早期的集群管理方式。

如果对外暴露的服务不多，我觉得仅仅配置 Basic Auth 并使用 [Bitwarden](https://lemondouble.github.io/zh-cn/p/fly.io-%EC%86%8C%EA%B0%9C-%EB%B0%8F-fly.io%EC%97%90-%EC%98%AC%EB%A6%AC%EA%B8%B0-%EC%A2%8B%EC%9D%80-%EC%84%9C%EB%B9%84%EC%8A%A4-%EC%B6%94%EC%B2%9C-vaultwarden/) 等密码管理器进行管理就已经足够了。

但如果以后项目越来越多呢？

为应对这种情况，我会在下一篇文章中介绍 SSO 认证方式。

但由于认证服务器是我自己 DIY 的，可能很难提供完整的源代码，请先理解我先介绍可直接使用的 Basic Auth 🙇
