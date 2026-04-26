---
title: "Introducing Fly.io and a Service Worth Hosting on It (VaultWarden)"
slug: 740c42a214d8ce6b
date: 2023-01-17T14:03:50Z
image: cover.png
categories:
    - SaaS
    - Fly.io
tags:
    - Self-hosting
    - Vaultwarden
    - Bitwarden
    - Password Manager
    - Docker
    - Fly.io
---

_Note: I'm based in Korea, so some context here is Korea-specific._

Fly.io is a company that lets you easily deploy a Docker image to a specific country (or region) — all you need to do is prepare the Docker image.

After signing up, once you write a `fly.toml` file appropriately and run

```bash
flyctl deploy
```

it does all the heavy lifting: uploading the image to the server and handling DNS, HTTPS setup, IP allocation, and monitoring (Grafana) configuration for you.

It also lets you scale up with just a few clicks. You could set all this up yourself, but Fly.io takes care of a lot of the tedious work, which makes things very convenient.

On top of that, even the free tier offers the following spec.

![](image-9.png)


But since it's a free tier, the spec isn't exactly generous.
So it's best suited for "applications where data management is important enough that you want to use a cloud service, but that don't consume a lot of compute resources."

I'm going to write about installing Vaultwarden, a password manager that fits exactly that use case.


### VaultWarden (a Bitwarden-compatible server)

![](image-10.png)

Bitwarden is a password manager that lets you share passwords across multiple devices via web browsers, Windows, iOS, and Android apps.

If you've ever used Chrome, the easiest analogy is the "Chrome autofill" feature.

On top of that, Bitwarden also supports OTP,

![Press the red button and you can authenticate without even opening the Google OTP app!](image-12.png)

a password generator,

![You can generate appropriate passwords matching each site's settings.](image-13.png)

and Organization-based password sharing.

This is handy when you're developing with others and need to share things like Slack Webhook URLs, environment variables, dev DB IDs and passwords, etc.

It also offers security audits — checking whether your password matches any leaked DB entries or whether you're reusing passwords — and a SEND feature for transmitting sensitive data with end-to-end encryption.

However, since Bitwarden is open-source, you can self-host the password server, but to ensure high availability at production level, it requires high specs and a lot of configuration. ([Bitwarden Install Docs](https://bitwarden.com/help/install-on-premise-linux/))

![The spec is a bit much to run on 256MB RAM..](image-14.png)


To solve this issue, there's a project called [Vaultwarden](https://github.com/dani-garcia/vaultwarden) which is compatible with Bitwarden Clients but with reduced spec requirements, using SQLite as the database so you can run the server with just a single Docker Image.


While the official server requires 2GB of RAM, this server uses about 100MB of memory at idle, which means deploying via Fly.io makes it really easy to use!

![RAM usage seen via Grafana after deploying Vaultwarden](image-14.png)


Let's follow the steps below to deploy a Vaultwarden server.
1. Follow Fly.io Docs ([Link](https://fly.io/docs/hands-on/install-flyctl/)) to install flyctl for your OS, and complete the login process.
2. Open a shell, create any folder (in my case I worked in `~/flyio/vaultwarden`), and move into it.
3. Run the command below to create a `fly.toml` config file. Change `app-name` to whatever you want. When prompted to choose a Region, I picked the closest one, Tokyo (nrt).

```bash
flyctl launch --name app-name --image vaultwarden/server:latest --no-deploy
```

4. Run the command below to create a Volume that will persistently store our data. Think of a Volume like an SSD that keeps the files alive even if the container dies. Match the app name with what you set in step 3, and similarly choose the closest Tokyo (nrt) region.

```bash
flyctl volumes create vaultwarden_data --size 1 --app app-name
```

5. Use the command below to create the Admin Token Secret that will be injected when the application starts. Think of `ADMIN_TOKEN` as the password for accessing the admin page.
Pick a separate password so it doesn't leak. We add this Secret value so we can enter the admin page for initial setup.

```bash
flyctl secrets set ADMIN_TOKEN=abc123456 
```


6. Edit the `fly.toml` file in the folder, referencing the file below.
You only need to modify the `[env]` section, the `[mounts]` section, and the `internal_port` field under `[[services]]` in the existing file.

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


7. From that folder, run the command below to deploy the app.

```bash
flyctl deploy
```

8. After deploying, run the command below to access the web page. Then click "Create Account" to create an account and a master password.

```bash
flyctl open
```

9.  (Optional) Use the command below to enter the admin page, and log in with the `ADMIN_TOKEN` you created above. Then configure any settings you need. In my case, I disallowed new sign-ups and turned off the invite feature.

```bash
flyctl open /admin
```


10. Then download the Bitwarden client, click the arrow at the top left, and enter the address of your deployed application. You can enter `app-name.fly.dev` or the same web page you just visited.

![](image-16.png)

Congratulations! If you've made it this far, your setup is complete!

From here on, you can enjoy a fairly convenient password manager.

Appendix 1. How do I add OTP?
When adding an ID/Password, paste the Secret Key shown below into the TOTP field.

![](image-17.png)
![](image-18.png)


Just enter a Secret Key like the one above.

Appendix 2. Can I use the Google passwords I'm already using?

A. Yes, you can!

After exporting your passwords from the Google Password Manager,

![](image-19.png)

go to Vaultwarden Web Console -> Login -> Tools -> Import Data, and you can keep using the passwords you had in Chrome as-is.

![](image-20.png)
