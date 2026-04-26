---
title: "通过 GitHub OIDC,无需在 GitHub Actions 中放置 AWS Key 即可使用 AWS 资源(免费)"
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
    - 安全
---

_注:作者居住在韩国,部分内容包含韩国特有的背景。_

要在 GitHub Actions 中使用 AWS,通常需要经过以下步骤。

- 进入 IAM 控制台,创建一个新用户。
- 为 GitHub Actions 添加 IAM Role,采用 Programmatic access 方式,获取 `AWS_ACCESS_KEY_ID` 和 `AWS_SECRET_ACCESS_KEY`。
- 在仓库设置中,将刚刚获取的 `AWS_ACCESS_KEY_ID` 和 `AWS_SECRET_ACCESS_KEY` 添加到 GitHub Secrets。
- 然后在 GitHub Actions 中添加如下工作流,即可访问 AWS 资源。

```yaml
- name: Configure AWS Credentials
    uses: aws-actions/configure-aws-credentials@v1
    with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: us-east-2
```

如果只有一个项目,这样管理也没什么问题。

但当使用 AWS 的项目数量开始增加时,就会遇到一些烦恼。

例如,是给每个仓库都发一个 IAM User?还是发一个超级用户,然后 `ACCESS_KEY` 和 `ACCESS_SECRET` 大家轮流用?

发超级用户的话,一旦相关密钥泄露就可能收到一张巨额的 AWS 账单,所以这并不是好的实践。

但要发多个用户的话,随着用户数量增加,又很难管理哪个 Key 用在了什么地方。

最近 AWS 增加了为每个 Access Key 添加 Description 的功能,管理稍微好了一些,但在那之前,因为不知道某个 Key 用在哪里而不敢动它的情况屡见不鲜。

![](2023-01-22-11-50-12.png)

再进一步!如果想遵循 Access Key 管理的最佳实践,就更让人头疼了。

![](2023-01-22-11-45-30.png)

按照最佳实践,需要定期轮换长期凭证,但一想到要定期获取新 Key 并去仓库里替换 Secrets,头就开始疼。

(大家拿到的不会过期的 `ACCESS_KEY`、`ACCESS_SECRET` 就被称为长期凭证。)

**如果不需要每次都把 Key 塞到 Secrets 里,而是自动只把我的 Key 塞到我的仓库里,那该多方便啊?**

而且,

**如果每次 Actions 运行时都签发一个只能短时间使用的临时 Token,即使 Token 被泄露了,过段时间也会过期失效,那该多好啊?**

通过 OIDC 连接就可以做到!而且,这也是 AWS 推荐的安全最佳实践([Link](https://github.com/aws-actions/configure-aws-credentials#assuming-a-role))!

![](2023-01-22-11-56-23.png)

下面介绍一种只需设置一次,以后跑 Actions 就会变得相当方便的方法!


* 配置方式有官方文档。如果遇到卡住的地方,请参考官方文档([Link](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services))

### 1. 创建 OIDC 身份提供者

* 进入 IAM Console。https://console.aws.amazon.com/iam/
* 在左侧菜单中选择 **身份提供商**,然后选择 **添加提供商**。
* 按以下方式设置。
    * 提供商 URL:`https://token.actions.githubusercontent.com`
    * 受众:`sts.amazonaws.com`


![](2023-01-22-12-03-09.png)

* 之后点击获取指纹,然后添加提供商。

### 2. 创建 OIDC 关联的 Role

* 选择 **角色** 菜单,创建一个新的角色。
* 选择 **Web 身份**,如下所示选择步骤 1 中创建的 GitHub Provider。

![](2023-01-22-12-08-13.png)


* 关于权限,这里只是简单示例,所以使用了 Managed Policy 中的 `S3FullAccess`。
    * 实际使用时,也可以自己创建 Policy 来使用。

![](2023-01-22-12-12-47.png)

* 填入合适的名称和说明,创建 Role。

![](2023-01-22-12-36-18.png)

* 创建完成!之后请把该 Role 的 ARN 记录在某处。

![](2023-01-22-12-38-36.png)

### 3. 在 GitHub Actions 中使用该 Role

* 可以按以下方式使用刚刚创建的 Role。
* 新建一个仓库,然后创建 `.github/workflows/oidc-connect-test.yaml`。
* **完全不需要设置 Access key 或 Secret 等内容!**

* 下面是一个将当前仓库原样上传到 S3 的 GitHub Actions。

* 请按自己的情况修改以下部分。
    * 将 `role-to-assume` 部分改为步骤 2 中记录的 Role 的 ARN。
    * 为了测试,随便创建一个名字的 bucket,并将 `<bucket 名称>` 部分替换为创建的 bucket 名称。

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
          role-to-assume: arn:aws:iam::123456461:role/github-actions-role # 使用刚才创建的 Role
          aws-region: ap-northeast-1

      - name: Copy file to S3 
        run: |
          aws s3 sync . s3://<bucket 名称>
```

* 之后,如果 Actions 正常运行

![](2023-01-22-12-42-54.png)

* 并且仓库文件实际被上传到了 S3,那么设置就成功了!

![](2023-01-22-12-43-30.png)


### Appendix 1. 增强安全性

* 因为是个人使用,目前所有仓库都可以获取 Credentials,但也可以仅限定特定仓库。
* 创建 Role 时,如下所示添加 repo,就可以仅在 `octo-org/octo-repo` 的所有分支/PR 中使用 AWS。

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
                    "token.actions.githubusercontent.com:sub": "repo:octo-org/octo-repo:*" # 添加这一部分
                },
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                }
            }
        }
    ]
}

```

### Appendix 2. Role 怎么管理?

* 已经做过一次设置后,以后再需要新的 Role,只需重新执行步骤 2(创建 OIDC 关联的 Role)创建新的 Role,然后只把 `role-to-assume` 部分换掉即可。

完!
