---
title: "GitHub OIDCを使ってGitHub ActionsにAWS Keyを入れずにAWSリソースを利用する(無料)"
date: 2023-01-22T11:18:50+09:00
image: cover.png
categories:
    - AWS
    - Github
tags:
    - AWS
    - OIDC
    - IAM
    - GitHub Actions
    - CI/CD
    - セキュリティ
---

_注：筆者は韓国在住のため、本文には韓国特有の文脈が含まれることがあります。_

GitHub ActionsからAWSを利用する場合、通常は次のような手順を踏みます。

- IAMコンソールに入って新しいユーザーを発行します。
- GitHub Actions用にIAM Roleを追加し、Programmatic access方式で `AWS_ACCESS_KEY_ID`、`AWS_SECRET_ACCESS_KEY` を発行します。
- リポジトリの設定から、GitHub Secretsに今発行した `AWS_ACCESS_KEY_ID`、`AWS_SECRET_ACCESS_KEY` を追加します。
- その後、GitHub Actionsで次のようなワークフローを追加してAWSリソースにアクセスします。

```yaml
- name: Configure AWS Credentials
    uses: aws-actions/configure-aws-credentials@v1
    with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: us-east-2
```

プロジェクトが一つだけなら、こうした管理でも大きな問題はありません。

しかしAWSを利用するプロジェクトが増え始めると、いくつかの悩みが出てきます。

例えば、リポジトリごとに一つのIAM Userを発行するのか?それともスーパーユーザーを一つ発行して `ACCESS_KEY`、`ACCESS_SECRET` を使い回すのか?

スーパーユーザーを発行すると、万一そのキーが流出した場合に大きなAWSの請求書を受け取りかねないため、良いプラクティスとは言えません。

かといって複数のユーザーを発行するとなると、ユーザー数が増えるに従ってどのキーがどこで使われているかを把握しづらくなる問題があります。

最近はAccess Keyごとに説明(Description)を付けられる機能が追加されて管理は少し楽になりましたが、それ以前はどのキーがどこで使われているかわからず、触れずに放置することがよくありました。

![](2023-01-22-11-50-12.png)

さらに一歩進んで!Access Key管理のベストプラクティスを守ろうとすると、もっと頭が痛くなります。

![](2023-01-22-11-45-30.png)

ベストプラクティスによれば、長期認証情報は定期的にローテーションする必要がありますが、定期的に新しいキーを発行してリポジトリのSecretsを差し替えることを考えると気が重くなります。

(皆さんが受け取る、有効期限のない `ACCESS_KEY`、`ACCESS_SECRET` を長期認証情報と呼びます。)

**毎回Secretsにキーを差し込まなくても、自分のリポジトリにだけ自動でキーを差してくれたら、どれほど楽でしょうか?**

また、

**Actionsが回るたびに短時間だけ使える一時トークンを発行して、万一トークンが漏れても時間が経てば期限切れで使えなくなったら、どれほど安心でしょうか?**

これはOIDC連携で実現できます!しかも、AWSが推奨するセキュリティのベストプラクティス([Link](https://github.com/aws-actions/configure-aws-credentials#assuming-a-role))でもあります!

![](2023-01-22-11-56-23.png)

一度設定しておけばActionsの運用がかなり楽になる方法をご紹介します!


* 設定方法は公式ドキュメントがあります。詰まる箇所があれば公式ドキュメントを参照してください([Link](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services))

### 1. OIDC ID プロバイダーの作成

* IAM Consoleにアクセスします。https://console.aws.amazon.com/iam/
* 左側メニューから **IDプロバイダー** を選択し、**プロバイダーを追加** をクリックします。
* 次のように設定します。
    * プロバイダーのURL: `https://token.actions.githubusercontent.com`
    * 対象者: `sts.amazonaws.com`


![](2023-01-22-12-03-09.png)

* その後、サムプリントを取得してプロバイダーを追加します。

### 2. OIDC連携用のRoleを作成

* **ロール** メニューを選択して新しいロールを作成します。
* **ウェブIDフェデレーション** を選択し、次のように1で作成したGitHub Providerを選択します。

![](2023-01-22-12-08-13.png)


* 権限については、今回は簡単な例なのでManaged Policyの `S3FullAccess` を使いました。
    * 実際に運用する際は、自分でPolicyを作成して使うこともできます。

![](2023-01-22-12-12-47.png)

* 適当な名前と説明を入れてRoleを作成します。

![](2023-01-22-12-36-18.png)

* 作成完了です!作成したRoleのARNをどこかに記録しておきましょう。

![](2023-01-22-12-38-36.png)

### 3. GitHub Actionsで該当のRoleを利用する

* 次のように、先ほど作成したRoleを利用できます。
* 新しいリポジトリを作成して `.github/workflows/oidc-connect-test.yaml` を作成します。
* **Access keyやSecretなどは一切設定する必要がありません!**

* 以下は、現在のリポジトリをそのままS3にアップロードするGitHub Actionsです。

* 次の部分を自分の環境に合わせて変更します。
    * `role-to-assume` の部分を、2.で記録したRoleのARNに置き換えます。
    * テスト用にバケットを適当な名前で一つ作成し、`<バケット名>` の部分を作成したバケット名に置き換えます。

```yaml
name: OIDC Connect Test

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    name: OIDC Connect Test
    runs-on: ubuntu-latest
    # These permissions are needed to interact with GitHub's OIDC Token endpoint.
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Configure Github Actions AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: arn:aws:iam::123456461:role/github-actions-role # 先ほど作成したRoleを使用
          aws-region: ap-northeast-1

      - name: Copy file to S3 
        run: |
          aws s3 sync . s3://<バケット名>
```

* その後、Actionsが正常に動作し

![](2023-01-22-12-42-54.png)

* 実際にS3にリポジトリのファイルがアップロードされていれば、設定成功です!

![](2023-01-22-12-43-30.png)


### Appendix 1. セキュリティの強化

* 個人利用のため現在は全てのリポジトリがCredentialsを取得できますが、特定のリポジトリだけに限定することもできます。
* Role作成時に下記のようにrepoを追加すると、`octo-org/octo-repo` の全てのブランチ/PRからのみAWSを利用できるようにできます。

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::123456123456:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": "repo:octo-org/octo-repo:*" # この部分を追加
                },
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                }
            }
        }
    ]
}

```

### Appendix 2. Roleはどう管理しますか?

* 一度セットアップしておけば、今後新しいRoleが必要になった際は2(OIDC連携用のRoleを作成)の手順で新しいRoleを作成し、`role-to-assume` の部分だけ差し替えればOKです。

以上!
