---
title: "在家用树莓派集群搭建数据中心 - 10-2. 使用 Forward Auth 和 OAuth2 实现 SSO"
slug: a63eb8199e2089b6
date: 2024-01-07T16:05:04+09:00
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
    - Forward Auth
    - OAuth2
    - SSO
    - 认证
---

_注：作者居住在韩国，部分内容包含韩国特有的背景。_

### 1. Forward Auth?

当我们需要认证逻辑时，可以在哪里实现认证逻辑呢？

最先想到的方法，可能是直接在应用中实现。

但如果应用变多了呢？或者部分应用并不是我自己开发的，而是别人开发的，我没法随意修改其中的认证逻辑，那该怎么办？

我们正在使用 Kubernetes，所有请求都会经过 Traefik 一次。

那么，让 Traefik 在`"转发请求之前"`处理认证逻辑如何？

![Traefik forward Auth 图](image.png)

**很简单！**

1. 所有请求都会经过 Traefik。
2. 如果配置了 Forward Auth，则向外部认证服务发送请求。
3. 外部认证服务执行如下操作：
   - 如果是有效请求，则返回 2XX OK 代码。这种情况下请求会正常继续。
   - 如果是无效请求，则原样返回认证服务器的响应。（返回错误）

那么我们怎么用这个呢？

### 2. OAuth2 与 Cookie、Forward Auth

#### 2-1. 最简单的 Forward Auth 服务器

让我们从头慢慢思考。

首先，思考一下最简单的 Forward Auth 服务器。

要让某些请求通过，某些请求被拒绝，应该怎么做？

没错！正如我们在 Basic Auth 时见过的那样，每个请求都附带认证信息，如果该认证信息有效就让它通过即可。

画成图的话，最简单的例子如下。

![最简单的 Forward Auth](image-1.png)

#### 2-2. 认证 Token 存放在哪里？

很好！

我们已经想出了最简单的 Forward Auth Server。那么接下来的问题是："那 Token 什么时候、在哪里获取？"

简单的方式可以是创建一个登录页面，登录成功后发放 Token。

画成图看看。

![增加登录页面](image-2.png)

咦？但是如果在`auth.lemondouble.com`登录，怎么共享 Token 呢？应该把 Token 存放在哪里？

而且我还需要在 a.lemondouble.com、b.lemondouble.com 等其他站点也使用该 Token 来保持登录。

（实际上这种情况其实挺常见的）如果在 a.lemondouble.com 已经完成 SSO 登录，但 b.lemondouble.com 又让我重新登录的话，（即使用户信息共享）也会非常麻烦！

为了解决这个问题，我们打算把 Token 存到 Cookie 里。

例如，在`.lemondouble.com`上设置 Cookie，那么这个 Cookie 会在访问所有子域名，也就是`a.lemondouble.com`、`auth.lemondouble.com`、`'b.lemondouble.com'`时自动一起发送。我们就利用这一点。

```javascript
res.cookie('forward-auth-token', token, {
    domain: '.lemondouble.com',        //subdomain까지 공유하려면 앞에 "."을 붙여줘야함!
});
```

这样一来，许多事情都会变得简单！整理一下。

1. Forward Auth 在有 Token 时允许，没有 Token 时拒绝。
2. 创建登录页面，登录页面在登录成功时给`.lemondouble.com`设置 Cookie。

整理到这里，可以画出如下示意图。

![登录之后到保存 Cookie 的图](image-3.png)

#### 2-3. 使用 JWT Token 及过期、有效性验证

那么！Token 里面应该放什么值好呢？

这种情况下，JWT 看起来比较合适。无状态、且非常难以篡改的 Token 正好适合这种用例。

里面的内容嘛……用户 ID 或邮箱大概比较合适。

但是 Token 一旦发放就能用一辈子吗……？那么 Token 一旦被盗，服务器就一辈子被攻陷了……？所以需要设置 Token 有效期。要在安全和便利之间权衡，为了方便，我们假设使用 3600 秒（1 小时）。

那么

1. JWT 的 Exp（Expired Time）应该由登录服务器设为当前时间 + 3600 秒发放。为了方便，把 Cookie 保留时间也统一为 3600 秒比较好。
2. 之前 Forward Auth 是只要有 Token 就一律放行，但现在需要增加 JWT 验证逻辑，并且即便值有效，过期 Token 也要拒绝。

再画一次图。

![增加 JWT Token 与过期机制](image-4.png)

#### 2-4. 重定向处理

但是如果 Token 无效，或者没有 Token，仅仅显示错误页面真的没问题吗？

当然……我自己用的话也没什么问题……但每次都会非常麻烦。如果认证失败时自动重定向到登录页面会更方便。

马上加上吧。

![增加重定向](image-5.png)

但是通常 SSO 成功后会回到原来的站点，对吧？这个也加上看看。

我们可以通过查询参数或 Header 等方式记住"原本是从哪里来的"，登录成功后登录服务器再根据这个记录做重定向处理就行了。

![回到原站点](image-6.png)

啊……OAuth 还没开始说就已经很复杂了。先再来一遍吧……

如果统一认证是 ID/Password 登录，那么到这里就完成了。这里可能有点容易混淆，整理一下：登录服务器和 Traefik 的 Forward Auth 服务器可以是同一个。

之所以画图重新写一次，是因为 Forward Auth 认证服务器和登录服务器没有必要分开！

![失败时流程重新整理的图](image-7.png)

#### 2-5. OAuth 开始！

先确定规格再继续吧。

我们要做的规格如下。

1. 如果访问某个服务，并且需要 OAuth SSO，则直接跳转到对应 Social Provider 的登录页面。（也就是说，我们不做登录页面！）
2. 登录成功后，重定向到原本要访问的服务！

至于 Client ID 注册等基本内容就省略了。一一说明的话能写成一本书……如果对 OAuth 2.0 协议不太了解的话，推荐生活编程的 [Web2 - OAuth 2.0](https://opentutorials.org/course/3405) 课程。[^1]（免费的！）

#### 2-6. OAuth 完整流程

我……真的努力想把这个讲简单一点……但说实话也想不到好办法……

直接给大家展示完整流程。

![OAuth SSO 完整流程](image-8.png)

核心思路是利用 State Code！

- 向 OAuth 认证服务器发送请求时，生成一个任意字符串一起发送（?state=任意字符串），之后用户登录成功返回时，会把该字符串一起送回来——这是 OAuth 2.0 标准中的规范。（不是必填参数，所以你可能在做 OAuth 集成时没注意到！）
- 利用这个机制，把用户"从哪里来"记录到数据库里，登录成功后再重定向到该 URL。

完整流程请参考上图。那么我们的认证服务器需要实现什么呢？

只需实现一个端点，但要实现以下逻辑。（按上面的顺序处理。处理顺序很重要！）

1. 如果查询参数中有 code 和 state，检查 State Code 是否与 DB 中保存的值一致。如果该 State Code 存在于 DB 中，则与第三方 OAuth Server 通信，把 Authorization Code 转换为用户信息。检查转换后的用户信息是否在预先注册的权限列表中，如果不在则返回错误。如果在权限列表中，生成新的登录 JWT，并以`.yourdomain.com`存到 Cookie 里。这时为了安全，最好开启 secure、httpOnly。另外 sameSite 选项设为`'Lax'`。然后将用户重定向到与 State Code 配对的 URI。

2. 如果 Cookie 中有 JWT Token，则在 JWT 验证后，如果是合法 JWT 则返回 200。


3. 如果 Cookie 中根本没有 JWT Token，或者是过期的 Cookie，则删除过期 Cookie，解析 `X-Forwarded-Host` 头和 `X-Forwarded-Uri` 来提取当前 URI。然后把新的 State Code 和该 URI 配对存入 DB，把当前 API 设为 Redirect URL，并连同 State Code 一起发送。

- 提示：用户请求的 URI：`https://" + GetHeader("X-Forwarded-Host") + GetHeader("X-Forwarded-Uri")`

#### 2-7. 追加配置

最后……在 Traefik 配置中加入以下两行。

```
- "--entryPoints.web.forwardedHeaders.insecure"
- "--entryPoints.websecure.forwardedHeaders.insecure"
```

没有这两行，X-forwarded-for 头就不会传过来！

如果你参考过我博客的文章（[4. Traefik HTTPS 认证及 ArgoCD 配置](/zh-cn/p/%EC%A7%91%EC%97%90%EC%84%9C-%EB%9D%BC%EC%A6%88%EB%B2%A0%EB%A6%AC-%ED%8C%8C%EC%9D%B4-%ED%81%B4%EB%9F%AC%EC%8A%A4%ED%84%B0%EB%A1%9C-%EB%8D%B0%EC%9D%B4%ED%84%B0%EC%84%BC%ED%84%B0-%EC%B0%A8%EB%A6%AC%EA%B8%B0-4.-traefik-https-%EC%9D%B8%EC%A6%9D-%EB%B0%8F-argocd-%EC%84%A4%EC%A0%95/)），可以这样配置。

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
      - "--entryPoints.web.forwardedHeaders.insecure"
      - "--entryPoints.websecure.forwardedHeaders.insecure"
```

之后执行 kubectl apply，完成 Forward Auth 配置！

### 结语

其实内容实在太庞大……我已经尽量写得简单了，但估计也没能很好地传达。以后大概要再修改一次这篇文章。

不过以防万一，如果有要参考着搭建的人，遇到不理解的地方，请随时在下方评论或通过 contact@lemondouble.com 联系我！我会一起帮忙排查问题。

啊，对了，如果你会 Golang，也可以参考 [traefik-forward-auth](https://github.com/thomseddon/traefik-forward-auth)。

[^1]: 生活编程（생활코딩）是韩国的一个热门在线学习平台，主要以韩语提供免费的编程课程。
