---
title: "Fly.io 介绍及适合部署到 Fly.io 的服务推荐（VaultWarden）"
slug: 740c42a214d8ce6b
date: 2023-01-17T14:03:50Z
image: cover.png
categories:
    - SaaS
    - Fly.io
tags:
    - 自托管
    - Vaultwarden
    - Bitwarden
    - 密码管理
    - Docker
    - Fly.io
---

_注：作者居住在韩国，部分内容包含韩国特有的背景。_

Fly.io 这家公司提供了一项功能：只要准备好 Docker 镜像，就能轻松地把镜像部署到指定国家等。

（注册完成后）只要适当地编写一个名为 `fly.toml` 的文件，然后输入

```bash
flyctl deploy
```

它就会帮你处理好一切，将镜像上传到服务器，并完成 DNS、HTTPS 设置、IP 分配、监控（Grafana）配置等。

而且只需点几下就能 Scale-up。虽然你也可以自己手动配置，但这些相当繁琐的工作它都会替你完成，非常方便。

此外，免费层也提供如下规格。

![](image-9.png)


不过既然是免费层，规格自然并不宽裕。
因此它适合那种"数据管理很重要、希望使用云服务，但又不会消耗太多计算资源"的应用。

下面我就来介绍如何安装一款非常契合此用途的密码管理器 Vaultwarden。


### VaultWarden（Bitwarden 兼容服务器）

![](image-10.png)

Bitwarden 是一款密码管理器，能让你通过网页浏览器、Windows、iOS、Android 应用等在多台设备之间共享密码。

如果你用过 Chrome，把它想成"Chrome 自动填充"功能就最容易理解。

除此之外，Bitwarden 还支持 OTP 功能，

![点击红色按钮，无需打开 Google OTP 应用即可完成认证！](image-12.png)

并提供密码生成器，

![可以根据各个网站的设置生成合适的密码。](image-13.png)

以及通过 Organization（组织）设置共享密码的功能。

如果你和别人一起开发，需要共享 Slack Webhook URL、环境变量、开发数据库 ID 与密码等，这些功能就非常实用。

此外它还提供安全审计功能，可以检查被泄露的数据库中是否有与你密码一致的项、是否存在被重复使用的密码等，以及通过端到端加密发送敏感数据的 SEND 功能等。

然而，Bitwarden 虽然是开源的，可以 Self-hosting 密码服务器，但要在生产级别上保证高可用性，需要较高的规格和不少配置。（[Bitwarden Install Docs](https://bitwarden.com/help/install-on-premise-linux/)）

![要在 256MB RAM 上运行，配置就有点……](image-14.png)


为了解决这个问题，有个名为 [Vaultwarden](https://github.com/dani-garcia/vaultwarden) 的项目，它兼容 Bitwarden Client，但降低了规格要求，并采用 SQLite 作为数据库，仅靠一个 Docker Image 就能跑起服务器。


相比官方服务器需要 2GB RAM，这个服务器在 Idle 状态下约只需 100MB 内存，所以利用 Fly.io 部署后就能轻松使用！

![Vaultwarden 部署后通过 Grafana 查看的 RAM 使用量](image-14.png)


让我们按照下面的步骤来部署一台 Vaultwarden 服务器吧。
1. 按照 Fly.io Docs（[Link](https://fly.io/docs/hands-on/install-flyctl/)）中的说明，安装与你的 OS 匹配的 flyctl，并完成登录。
2. 在 Shell 中创建任意一个文件夹（我使用的是 `~/flyio/vaultwarden` 文件夹），然后进入该文件夹。
3 . 输入下面的命令创建一个名为 `fly.toml` 的配置文件。此时，`app-name` 可以按需修改。输入时会弹出选择 Region 的窗口，我选择了距离最近的 Tokyo(nrt) 区域。

```bash
flyctl launch --name app-name --image vaultwarden/server:latest --no-deploy
```

4. 输入下面的命令，创建一个用于持久化保存数据的 Volume。Volume 你可以理解为一种 SSD：即使容器挂掉，存储在其中的文件也不会丢失。app 名称要与第 3 步中设置的 app 名一致，Region 同样选择最近的 Tokyo(nrt) 区域。

```bash
flyctl volumes create vaultwarden_data --size 1 --app app-name
```

5. 使用以下命令创建应用启动时注入的 Admin Token Secrets。`ADMIN_TOKEN` 你可以理解为进入管理员页面所需的密码。
请使用一个不会泄露的独立密码；为了进入 admin 页面进行初始设置，我们会添加这个 Secrets 值。

```bash
flyctl secrets set ADMIN_TOKEN=abc123456 
```


6. 参考下面的文件，修改文件夹中的 `fly.toml` 文件。
只需在原有文件中修改 `[env]` 部分、`[mounts]` 部分以及 `[[services]]` -> `internal_port` 部分即可。

```yml
app = "app-name"
kill_signal = "SIGINT"
kill_timeout = 5
processes = []

[env]
  ROCKET_PORT = 8080

[experimental]
  allowed_public_ports = []
  auto_rollback = true

[mounts]
  destination = "/data"
  source = "vaultwarden_data"

[[services]]
  http_checks = []
  internal_port = 8080
  processes = ["app"]
  protocol = "tcp"
  script_checks = []
  [services.concurrency]
    hard_limit = 25
    soft_limit = 20
    type = "connections"

  [[services.ports]]
    force_https = true
    handlers = ["http"]
    port = 80

  [[services.ports]]
    handlers = ["tls", "http"]
    port = 443

  [[services.tcp_checks]]
    grace_period = "1s"
    interval = "15s"
    restart_limit = 0
    timeout = "2s"
```


7. 在该文件夹下执行下面的命令部署应用。

```bash
flyctl deploy
```

8. 部署后，输入下面的命令访问网页。然后点击"创建账号"，进行账号创建和主密码创建。

```bash
flyctl open
```

9.  (Optional) 之后用下面的命令进入管理员页面，并使用上面创建的 `ADMIN_TOKEN` 登录。如有需要的设置可以进行配置。我自己关闭了新用户注册，并关闭了邀请功能。

```bash
flyctl open /admin
```


10. 然后下载 Bitwarden 客户端，点击左上角的箭头，输入你部署好的应用地址。可输入 `app-name.fly.dev`，或者输入你刚刚访问过的网页地址。

![](image-16.png)

恭喜你！ 走到这一步说明设置已经完成！

之后就可以享用一款相当好用的密码管理器了。

Appendix 1. OTP 怎么添加？
在添加 ID/Password 时，在如下的 TOTP 栏中

![](image-17.png)
![](image-18.png)


输入上面那样的 Secret Key 即可。

Appendix 2. 我已经在用 Google 密码，可以接着用吗？

A. 可以！

从 Google 密码管理器中导出密码后，

![](image-19.png)

进入 Vaultwarden 网页控制台 -> 登录 -> 工具 -> 导入数据，就可以继续使用你之前在 Chrome 中使用的密码。

![](image-20.png)
