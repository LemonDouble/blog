---
title: "Building a Home Datacenter with a Raspberry Pi Cluster - 8. Setting Up a Private Docker Registry with ID/Password Auth"
slug: be354d4b5f50a3ee
date: 2023-12-10T10:08:50+09:00
image: cover.png
categories:
    - K8S
    - K3S
    - Raspberry Pi
tags:
    - K8S
    - K3S
    - Raspberry Pi
    - Docker Registry
    - Container Registry
    - Longhorn
---

_Note: I'm based in Korea, so some context here is Korea-specific._

### 1. Introduction

Since we already have the infrastructure, let's go ahead and build our own Private Docker Registry too.

In my case, I run several personal servers on this cluster, so I use a Private Registry to manage those container images.

If you're willing to pay for Dockerhub or if Public Images are enough for you, you don't need to follow this guide.

Since we'll be using a Private Repo, we'll set up an ID and Password at the same time!

### 2. Installation

Let's install it quickly with ArgoCD again!

`modules/docker-registry-system/pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: longhorn-docker-registry-pvc
  namespace: docker-registry-system
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn-ssd
  resources:
    requests:
      storage: 300Gi
```

I run an ML server, so my image sizes are large, which is why I gave it a generous 300Gi. Adjust this to fit your needs.

`modules/docker-registry-system/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: registry-service
  namespace: docker-registry-system
spec:
  selector:
    app: registry
  type: LoadBalancer
  ports:
    - name: docker-port
      protocol: TCP
      port: 5000
      targetPort: 5000
  loadBalancerIP: 192.168.0.202
```

`modules/docker-registry-system/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: registry
  namespace: docker-registry-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: registry
  template:
    metadata:
      labels:
        app: registry
        name: registry
    spec:
      nodeSelector:
        node-type: worker
      containers:
      - name: registry
        image: registry:2
        ports:
        - containerPort: 5000
        env:
        - name: REGISTRY_HTTP_ADDR
          value: 0.0.0.0:5000
        - name: REGISTRY_AUTH
          value: htpasswd
        - name: REGISTRY_AUTH_HTPASSWD_REALM
          value: docker-registry
        - name: REGISTRY_AUTH_HTPASSWD_PATH
          value: /auth/registry.password
        - name: REGISTRY_STORAGE_DELETE_ENABLED
          value: "true"
        volumeMounts:
        - name: lv-storage
          mountPath: /var/lib/registry
          subPath: registry
        - name: docker-registry-account-htpasswd
          mountPath: /auth
          readOnly: true
      volumes:
        - name: lv-storage
          persistentVolumeClaim:
            claimName: longhorn-docker-registry-pvc
        - name: docker-registry-account-htpasswd
          secret:
            secretName: docker-registry-account-htpasswd
```

We specify the ID and Password the Registry will use through a Secret called docker-registry-account-htpasswd. I'll cover that next.

`modules/docker-registry-system/sealed-docker-registry-account-htpasswd.yaml`

The Secrets setup is a bit involved. Let's go through it step by step!

1. On the Control Node, run `htpasswd -Bbn <id> <password>` to get the id/password string the Docker Registry will use.
2. Use the kubeseal-webgui you set up in 7-1 to encrypt the Secret.
![Example](image.png)
1. Paste the resulting Secret into sealed-docker-registry-account-htpasswd.yaml.

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: docker-registry-account-htpasswd
  namespace: docker-registry-system
  annotations: {}
spec:
  encryptedData: 
    registry.password: dfadf...
```

`modules/docker-registry-system/ingress.yaml`
```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: docker-registry-ingress
  namespace: docker-registry-system
spec:
  tls:
    certResolver: le
  routes:
    - kind: Rule
      match: Host(`<subdomain to use>`)
      services:
        - name: registry-service
          port: 5000
```

### 3. Verifying It Works

Once you've confirmed everything is set up correctly, deploy it!

After deploying, check that you can log in successfully with the id/password.

On a machine with docker installed, run the following from the terminal.

```shell
docker login <my-registry.lemon.com>
```

Then enter the id/password and confirm the login goes through.

### 4. How to Pull Images from the Private Registry

Now we need to know how to pull images from a Private Registry that requires an Id/Password, right? Otherwise we can't actually use the images we've pushed.

I'm sharing the method I use. There's probably a cleaner way, so please let me know if you find one!

Suppose I want to use it in a namespace called auth-server-system:

1. Run `kubectl create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>` to create a secret called regcred in the default Namespace.
2. Run `kubectl get secret regcred --output=yaml > secret.yaml` to save the secret info into secret.yaml. Back this Secret up somewhere safe.
3. Run `kubectl delete secret regcred` to remove the secret you just created in the default namespace.
4. Now edit the Secret from step 2 and change its Namespace to the one you want to use (e.g., auth-server-system).

```yaml
apiVersion: v1
data:
  .dockerconfigjson: ddfadf...
kind: Secret
metadata:
  name: regcred
  namespace: auth-server-system # change this each time
type: kubernetes.io/dockerconfigjson
```

5. Then run `cat secret.yaml | kubeseal --controller-namespace=sealed-secrets-system --controller-name=sealed-secrets -oyaml > sealed-regcred.yaml` to generate a Sealed Secret.
6. Deploy the Sealed regcred together with your workload, and add an ImagePullSecret to your Deployment as shown below.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-server
  namespace: auth-server-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: auth-server
  template:
    metadata:
      labels:
        app: auth-server
        name: auth-server
    spec:
      nodeSelector:
        node-type: worker
      containers:
        - name: auth-server
          image: registry.lemon.com/auth_server_repository:2023.12.10.1
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 80
      imagePullSecrets:
        - name: regcred # add this
```

### 5. Wrapping Up

Congratulations! You've now finished setting up a Private Docker Registry,

and you can keep your images safe behind an Id/Password.

Next time, I'll walk through how to manage images via a Web UI, and cover Docker GC.
