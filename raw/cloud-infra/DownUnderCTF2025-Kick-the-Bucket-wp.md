# kick the bucket

## 题目简述

题目提供一个由 CI/CD 流水线用户生成的 S3 预签名 URL，以及对应的 bucket resource policy。预签名 URL 已携带访问对象所需的签名，但资源策略额外要求请求上下文中的 `aws:UserAgent` 匹配 `aws-sdk-go*`。关键问题是：`User-Agent` 完全由客户端提供，不应被当作可信授权属性。

## 解题过程

资源策略的决定性部分如下：

```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Principal": {
    "AWS": "arn:aws:iam::<account-id>:user/pipeline"
  },
  "Condition": {
    "StringLike": {
      "aws:UserAgent": "aws-sdk-go*"
    }
  }
}
```

S3 会先按预签名 URL 所代表的流水线身份评估权限，再检查策略条件。通配符 `*` 允许任意后缀，而 HTTP 客户端可以自由设置 `User-Agent`，因此只需让请求头以前缀 `aws-sdk-go` 开始：

```bash
curl --user-agent "aws-sdk-go-ductf" "<题目提供的预签名 URL>"
```

普通浏览器请求不满足条件时，S3 的拒绝信息会显示该 URL 代表的是流水线用户；伪造请求头后，同一个已签名请求通过 `StringLike` 条件并返回 `flag.txt`。长期 WP 不保留已经过期的具体预签名 URL，也不记录真实账号与 bucket 标识，因为它们不是漏洞成立的必要条件。

## 方法总结

- 核心技巧：沿 `identity → action → resource → condition` 拆解 S3 授权，并识别可由客户端伪造的条件键。
- 识别信号：策略用 `aws:UserAgent` 限制对象读取，同时题目已经给出预签名 URL；这意味着身份签名不是缺口，策略条件才是突破点。
- 复用要点：预签名 URL 不会让资源策略失效，但它也不会使客户端请求头变可信。授权条件应依赖不可伪造或由 AWS 控制的上下文，而不是 `User-Agent` 这类任意输入。
