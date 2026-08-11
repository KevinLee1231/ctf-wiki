# Mary had a little lambda

## 题目简述

题目泄露了 AWS `devopsadmin` 的访问配置。Terraform 定义的 Lambda `yakbase` 需要读取加密的 SSM 参数 `/production/database/password`，而其执行角色的信任策略又把 `devopsadmin` 列为可 `sts:AssumeRole` 的主体。初始用户拥有 Lambda 枚举与代码下载权限，角色拥有 SSM 读取权限；跨越两套权限后可请求解密的参数值。关键是 IAM 身份与资源授权图，归入 cloud-infra。

## 解题过程

先用题目提供的 profile 做最小只读确认，并枚举 Lambda：

```sh
aws sts get-caller-identity
aws lambda list-functions
aws lambda get-function --function-name yakbase
```

Terraform 显示 `devopsadmin` 被允许 `lambda:ListFunctions`、`lambda:GetFunction` 和 `iam:GetRole`；`GetFunction` 返回函数部署包的临时下载位置。只读取包内 Python 源码即可看到它使用的 SSM 参数名：

```text
/production/database/password
```

再读取 Lambda 角色的信任关系。配置中同时允许 `lambda.amazonaws.com` 与 `devopsadmin` 扮演该角色：

```hcl
principals {
  type        = "AWS"
  identifiers = [aws_iam_user.devopsadmin.arn]
}
```

因此用 Lambda 的 role ARN 取得短期 STS 凭据，并仅在当前 shell 中以该临时身份调用 SSM：

```sh
aws sts assume-role --role-arn "$FUNCTION_ROLE_ARN" --role-session-name lambda-access
aws ssm get-parameter \
  --name /production/database/password \
  --with-decryption
```

角色策略授予带 Challenge 标签资源的 `ssm:GetParameter`；参数类型是 `SecureString`，所以不带 `--with-decryption` 时只会得到密文。带上该选项后，`Parameter.Value` 即为 flag。题目未在仓库静态资料中保留实际部署期的参数值，故不将未验证的字面 flag 写入本文。

## 方法总结

- 核心技巧：分别画出初始用户和服务角色的权限，再使用错误配置的 trust policy 将“读函数源码”与“读解密参数”拼接。
- 识别信号：泄露访问 key、Serverless 函数、SSM `SecureString` 与可被非服务主体 assume 的 Lambda role 同时出现时，应优先检查身份到资源的授权图。
- 复用要点：先做 `GetCallerIdentity`、只读资源与策略查询；不要把长期访问 key 写入文档或全局配置。修复应移除管理员对运行角色的信任、采用最小权限且限定参数 ARN，并使用可轮换的短期凭据。
