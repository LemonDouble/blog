---
title: "Building a Home Datacenter with a Raspberry Pi Cluster - 10-1. Protecting Internal Services with Basic Auth"
slug: 771435ba12d3f38c
date: 2024-01-07T12:41:24+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - Raspberry Pi
tags:
    - K8S
    - K3S
    - Raspberry Pi
    - Traefik
    - Basic Auth
    - Sealed Secrets
    - Authentication
---

_Note: I'm based in Korea, so some context here is Korea-specific._

**The content below overlaps significantly with [Section 3](https://lemondouble.github.io/en/p/%EC%A7%91%EC%97%90%EC%84%9C-%EB%9D%BC%EC%A6%88%EB%B2%A0%EB%A6%AC-%ED%8C%8C%EC%9D%B4-%ED%81%B4%EB%9F%AC%EC%8A%A4%ED%84%B0%EB%A1%9C-%EB%8D%B0%EC%9D%B4%ED%84%B0%EC%84%BC%ED%84%B0-%EC%B0%A8%EB%A6%AC%EA%B8%B0-7.-sealed-secrets%EB%A5%BC-%ED%86%B5%ED%95%9C-%EB%B9%84%EB%B0%80-%EA%B4%80%EB%A6%AC--traefik-basic-auth-%EC%84%A4%EC%A0%95/#3-sealed-secrets-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC-basic-auth-%EB%93%B1%EB%A1%9D%ED%95%B4%EB%B3%B4%EA%B8%B0) of the Sealed Secrets post. If you've already gone through that section and are familiar with Basic Auth, feel free to skip.**

### 1. Introduction

When working with Kubernetes, you end up running quite a few services.

Of course, keeping things on the internal network only is the most secure option, but you might not always be able to manage your cluster from inside that network. For example, something could go wrong while you're traveling.

In such cases, there are roughly two ways to handle it.

1. Use a VPN to connect to the internal network from another computer and use the related services
2. Expose the service to the internet but add an authentication mechanism

I personally went with option 2 and built my own SSO server.

However, since setting up an SSO system is a fairly involved process, I want to start with the simplest authentication mechanism first.

### 2. The Flow of Basic Auth

![Basic Auth, source: MDN](image.png)

Basic Auth is super simple!

1. The client requests a page from the server.
2. The server receives the request, and if there's no authentication info, it responds with 401 and asks for an authentication method (Basic Auth).
3. The client receives the response and proceeds with Basic Auth.
    - At this point, it sends `Basic ${auth token}` in the HTTP Authorization header.
    - The auth token is Base64Encode("userId:password").
    - For example, if the id is lemondouble and the password is 1q2w3e4r!, you base64 encode `"lemondouble:1q2w3e4r!"` to get `bGVtb25kb3VibGU6MXEydzNlNHIh`.
4. The server verifies this token and returns a normal response.

### 3. Pros and Cons of Basic Auth

Pros:

1. Built-in! Most browsers support it without any extra setup.
2. Simple to implement. Most web frameworks support it out of the box!

Cons:

1. Weak security. This is the biggest downside.
   - The password is sent on every request, base64 encoded (= effectively plaintext).
     - It's fine if you use HTTPS, but with HTTP, the password is transmitted in plaintext on every request!
   - It doesn't defend against brute-force attacks.
     - This is more of an Id/Password issue, but with Traefik's default settings, there's no feature like dropping clients that have failed too many requests.
     - If your Id/Password is weak, a malicious attacker could gain access to the service via brute force.

Therefore, if possible, I recommend using this feature for:

1. Services where access isn't a big deal, or services you only want to share with a few users
2. Over the HTTPS protocol
3. With a sufficiently long password

### 4. How to Configure Basic Auth

- We'll use Sealed Secrets to encrypt access info (Id/Password) and store it in Git.
- The reasons are:
  - To recover the cluster quickly along with access info in case of an unexpected accident (since Secret info is also in Git, you just deploy)
  - To prevent accidental Id/Password exposure due to a momentary mistake, even when using a Private Repo

Disclaimer: The content below is identical to [Section 3](https://lemondouble.github.io/en/p/%EC%A7%91%EC%97%90%EC%84%9C-%EB%9D%BC%EC%A6%88%EB%B2%A0%EB%A6%AC-%ED%8C%8C%EC%9D%B4-%ED%81%B4%EB%9F%AC%EC%8A%A4%ED%84%B0%EB%A1%9C-%EB%8D%B0%EC%9D%B4%ED%84%B0%EC%84%BC%ED%84%B0-%EC%B0%A8%EB%A6%AC%EA%B8%B0-7.-sealed-secrets%EB%A5%BC-%ED%86%B5%ED%95%9C-%EB%B9%84%EB%B0%80-%EA%B4%80%EB%A6%AC--traefik-basic-auth-%EC%84%A4%EC%A0%95/#3-sealed-secrets-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC-basic-auth-%EB%93%B1%EB%A1%9D%ED%95%B4%EB%B3%B4%EA%B8%B0) of the Sealed Secrets post!

Let's learn how to configure Basic Auth by setting up the Longhorn Dashboard to be accessible from outside.

1. Run `apt install apache2-utils` so you can use the htpasswd command.
2. Use `htpasswd -nb <id> <password> | openssl base64` to get a Secret String containing the id and password.
3. Create secret.yaml with any text editor, and add a Secret based on the following:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: longhorn-system-basic-auth
  namespace: longhorn-system
data:
  users: <String obtained in step 2>
```

4. Run `cat secret.yaml | kubeseal --controller-namespace=sealed-secrets-system --controller-name=sealed-secrets -oyaml > sealed-secrets.yaml` to get sealed-secrets.yaml.
5. Register the ingress as follows:

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
      match: Host(`<your desired subdomain, e.g. longhorn.lemon.com>`)
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

In `modules/longhorn-system/sealed-basic-auth-secret.yaml`, paste the sealed-secrets.yaml you just generated.

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

6. Then deploy.
7. After that, when you access the page, an id/password prompt will appear, and you won't be able to see the page without logging in!

### 5. Wrapping Up

Basic Auth was my initial cluster management approach.

If you don't have many services exposed externally, I felt that just configuring Basic Auth and managing credentials with a password manager like [Bitwarden](https://lemondouble.github.io/en/p/fly.io-%EC%86%8C%EA%B0%9C-%EB%B0%8F-fly.io%EC%97%90-%EC%98%AC%EB%A6%AC%EA%B8%B0-%EC%A2%8B%EC%9D%80-%EC%84%9C%EB%B9%84%EC%8A%A4-%EC%B6%94%EC%B2%9C-vaultwarden/) was good enough.

But what if you end up with more projects later?

I'll cover an SSO authentication method in the next post to address that scenario.

That said, since the auth server is DIY-built, it'll be hard to provide the full source code. Please bear with me as I cover the more readily usable Basic Auth first.
