---
title: "用 AWS SSO 在家也能用上公司里那块熟悉的登录页（免费）"
slug: 1c9eaebbdee2ac98
date: 2023-01-21T16:17:06+09:00
image: cover.png
categories:
    - AWS
tags:
    - AWS
    - IAM Identity Center
    - SSO
    - AWS CLI
    - Organizations
---

_注：作者居住在韩国，部分内容包含韩国特有的背景。_

如果你在公司或组织里用过 AWS，应该见过下面这种 SSO（Single Sign On）登录页面。

![](2023-01-21-23-57-37.png)

一旦设置好，会话期间不用每次都输入 id/password，使用起来很方便。

而且在使用 aws-cli 或 aws-sdk 时，也不用把 `AWS_ACCESS_KEY`、`AWS_ACCESS_SECRET` 这类不能泄露的信息以明文形式存到本地电脑上，安全性也更好。

并且这是一项免费服务（[Link](https://aws.amazon.com/ko/iam/identity-center/faqs/)），**完全** 不会产生额外费用！

![](2023-01-22-00-05-31.png)

那我们来配置一下吧！


## 1. 配置 IAM Identity Center

### 1. 在 AWS 中搜索 SSO，找到 IAM Identity Center。

![](2023-01-22-00-07-58.png)

### 2. 点击启用！
- 此时，Identity Center 只能在一个区域中启用，因此在你常用的区域启用更便于后续管理。

![](2023-01-22-00-12-59.png)

### 3. 先配置 AWS Organization。

![](2023-01-22-01-00-14.png)

- 你会遇到上面这样的报错，需要先创建 AWS Organization。
- AWS Organization 同样是*免费*服务。（[Link](https://docs.aws.amazon.com/ko_kr/organizations/latest/userguide/orgs_introduction.html#pricing)）

### 4. 创建好 Dashboard 后，点击"前往设置"按钮进行后续配置。

![](2023-01-22-00-16-22.png)

### 5. 先给 SSO 登录页面起个别名吧。
 - 默认提供的登录页面像 `aadfbasdf-d.awsapps.com/start` 这样自动生成，难以记住。
 - 我们把它改成 `lemonlogin.awsapps.com/start` 这种好记的别名。
 - 只能设置一次，所以请慎重决定！

![点击"自定义 AWS 访问门户 URL"。](2023-01-22-00-18-34.png)

![在这里填上你想要的别名。](2023-01-22-00-20-02.png)

### 6. 添加群组！

![](2023-01-22-01-10-53.png)

添加群组。

### 7. 添加用户！

![](2023-01-22-00-22-44.png)

- 点击添加用户按钮后……

![](2023-01-22-00-24-25.png)

- 填写用户信息。密码可以通过邮件接收设置邮件，也可以使用一次性密码。
- 我们就用默认的邮件方式吧。

![](2023-01-22-01-12-26.png)

- 之后可以选择 Group，把第 6 步里设置的群组添加上。

- 然后查看邮件，接受邀请，给该账户设置 Password 并记录到某个地方！

### 8. 添加权限！

![](2023-01-22-00-51-29.png)

- 选择权限集。我选择的是 `AdministratorAccess`，你也可以选别的。
- 如果预定义的权限集中没有想要的 Role，也可以使用自定义来直接指定。

![](2023-01-22-00-53-34.png)

- 接着填写其余设置。
- 会话时长指的是过了这段时间后 AWS 自动登出的时间。
- 默认是 1 小时，频繁登出比较烦人，我把它延长到了 4 小时左右。
- `中继状态`可以设置登录成功后的重定向 URL。比如 Billing 权限就可以在登录成功后跳转到 Billing 页面。
- 如果有需要，可以参考文档（[Link](https://docs.aws.amazon.com/ko_kr/singlesignon/latest/userguide/howtopermrelaystate.html)）进行配置。我留空了。


### 9. 配置 SSO 登录

![](2023-01-22-01-30-57.png)

- 然后在 AWS 账户标签页里选择自己的账户，点击`分配用户或群组`。

![](2023-01-22-01-21-14.png)

- 把上面设置的群组加进来，

![](2023-01-22-01-21-43.png)

- 再把上面设置的权限加上。

### 10. SSO 登录

![](2023-01-22-01-33-05.png)

- 现在去第 5 步里设置的 AWS 登录地址登录吧。访问 `别名.awsapps.com/start` 进行登录。

![](2023-01-22-01-34-14.png)

- SSO 配置成功！


### Appendix 1. 加强安全

![](2023-01-22-01-37-55.png)

- 进入 IAM Identity Center - 设置，可以为 SSO 账户配置 MFA、设置会话长度等。
- 即使其他保持默认，也建议开启`如果该用户没有已注册的 MFA 设备 - 在登录时要求注册 MFA 设备`这一选项。

### Appendix 2. 通过 AWS SSO 使用 AWS_CLI

![](2023-01-22-01-47-10.png)

- 完成 SSO 配置后，按上图使用 `aws configure sso` 配置登录。（红色下划线的行是手动输入的。）
- 配置完成后，使用 `aws s3 ls` 等命令确认配置是否正常。
