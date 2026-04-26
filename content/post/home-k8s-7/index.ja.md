---
title: "自宅でラズベリーパイクラスタによるデータセンター構築 - 7. Sealed Secretsによる秘密管理 + Traefik Basic Auth設定"
slug: 1379182f14a62e9b
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
    - 秘密管理
    - kubeseal
    - Traefik
---

_注：筆者は韓国在住のため、本文には韓国特有の文脈が含まれることがあります。_

### 1. Sealed Secretsとは?

私たちは今GitOpsを行っています。つまり、Gitにクラスタに関連するすべての設定をアップロードしています。

ところが、KubernetesのSecretには暗号化機能がありません。base64でデコードすれば誰でも平文で内容を見ることができます。

それなら、Gitにアップロードできませんよね?

このような問題を解決するために登場したのがSealed Secretsです。

1. クラスタは暗号化/復号鍵を持ちます。
2. ユーザーはGitにファイルをアップロードする前に、そのクラスタが持っている鍵で暗号化を行います。
3. Gitに暗号化されたSecretがアップロードされます。誰でも見ることはできますが、復号は不可能です。したがって安全です。
4. その後、ArgoCDなどでクラスタにそのSecretを再度読み込ませると、Sealed Secretコントローラが暗号化を解いて自動的に復号してくれます。

Sealed Secrets GitHub : [Link](https://github.com/bitnami-labs/sealed-secrets)

### 2. Sealed Secretsを追加する

ArgoCDを使ってあっという間にデプロイしてしまいましょう!

`apps/enabled/sealed-secrets-system.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sealed-secrets-system
  namespace: argocd
spec:
  destination:
    namespace: sealed-secrets-system
    server: 'https://kubernetes.default.svc'
  source:
    path: modules/sealed-secrets-system
    repoURL: 'git@github.com:<YourOrganizationName>/<YourRepositoryName>.git'
    targetRevision: HEAD
  project: default
```

`modules/sealed-secrets-system/sealed-secrets.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sealed-secrets
  namespace: argocd
spec:
  destination:
    namespace: sealed-secrets-system
    server: 'https://kubernetes.default.svc'
  source:
    repoURL: 'https://bitnami-labs.github.io/sealed-secrets'
    targetRevision: 2.13.3
    chart: sealed-secrets
    helm:
      releaseName: sealed-secrets
  project: default
```

その後、デプロイを進めます。

次に、以下のコマンドを使ってkubesealコマンドラインツールをインストールします。

```shell
KUBESEAL_VERSION='0.23.0' #2023.12時点の最新バージョン
wget "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION:?}/kubeseal-${KUBESEAL_VERSION:?}-linux-arm64.tar.gz"
tar -xvzf kubeseal-${KUBESEAL_VERSION:?}-linux-arm64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```


`secret.yaml`
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
  namespace: mynamespace
data:
  users: bGVtb246JGFwcjEkL0ZYczZGam0kMkpJZWNlQy45UWc5QjV5NUw2TzVoMAoK
```

以下のようなコマンドで、Sealed Secretsを作成して登録できます。
```sh
cat secret.yaml | kubeseal --controller-namespace=kube-sealed-secrets-system --controller-name=sealed-secrets -oyaml > sealed-secrets.yaml
```

### 3. Sealed Secretsを利用してBasic Authを登録してみる

Basic Authは、めちゃくちゃ簡単に言うとID/パスワードログインです。

![こういうやつ (Id, passwordログイン)](image.png)

Longhorn Dashboardを外部インターネットからもアクセスできるようにIngressを設定し、Id/パスワードログインを付けてみましょう! (Control01で実施)

1. `apt install apache2-utils`を実行して、htpasswdコマンドを使えるようにします。
2. `htpasswd -nb <id> <password> | openssl base64`を使って、id, passwordが入ったSecret Stringを取得します。
3. 任意のテキストエディタでsecret.yamlを作成し、以下を参考にSecretを追加します。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: longhorn-system-basic-auth
  namespace: longhorn-system
data:
  users: <2で取得したString>
```

4.`cat secret.yaml | kubeseal --controller-namespace=sealed-secrets-system --controller-name=sealed-secrets -oyaml > sealed-secrets.yaml`を入力して、sealed-secrets.yamlを取得します。
5. 以下を参考にしてingressを登録します。

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
      match: Host(`<希望するsubdomain、例:longhorn.lemon.com>`)
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

`modules/longhorn-system/sealed-basic-auth-secret.yaml`には、たった今生成したsealed-secrets.yamlをコピー&ペーストしましょう。

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

6. その後、デプロイを進めます。
7. アクセスすると、id/password確認画面が正常に表示され、ログインしないとそのページが見えなくなります!

### 3. バックアップとリカバリ

これで私たちはGitにSecretのような機密情報も保存できるようになりました!

ただ、いいことづくめではあるのですが、クラスタが壊れて復号鍵を失ってしまうと、私たち自身も秘密を解けなくなってしまいますよね?

したがって、暗号化/復号鍵は**必ず**バックアップしなければなりません!

- バックアップ方法

1. `kubectl get secret -n sealed-secrets-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml >master.key`を入力してmaster.keyをどこかに必ずバックアップしておきます。

- リカバリ方法

1. 先ほどバックアップしたmaster.keyをクラスタにコピーし、`kubectl apply -f master.key`を入力してmaster keyを登録します。
2. sealed-secrets-systemに存在するPodを1つkillして新しいKeyを読み込ませます。(ArgoCDから簡単に行えます)

### 4. おわりに

これで私たちはSealed Secretsを通じてGitにすべてのデータを保存できるようになり、

Traefik Basic Authを利用して内部サービスを外部に(最低限のセキュリティを備えて)公開できるようになりました!

それでは次回は、Private Docker Registryを構築する方法を見ていきます!
