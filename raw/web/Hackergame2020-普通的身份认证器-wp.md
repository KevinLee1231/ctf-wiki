# Hackergame2020 普通的身份认证器 WP

## 题目简述

应用用 RS256 签发 JWT，`/profile` 只向 `sub=admin` 的身份返回 flag。FastAPI 自动文档暴露了 `/debug`，该接口又泄露 RSA 公钥；服务端使用存在算法混淆缺陷的 PyJWT 1.5.0 验证令牌。

## 解题过程

访问 `/docs` 可以看到未出现在前端页面中的 `/debug` 路由，并取得 PEM 格式公钥。源码签发令牌时使用：

```python
jwt.encode(payload, PRIVATE_KEY, algorithm="RS256")
```

但验证时只调用 `jwt.decode(token, PUBLIC_KEY)`，允许令牌头自行选择算法。构造新的 payload：

```json
{"sub":"admin","exp":9602085613}
```

将 JWT 头部改成 `{"alg":"HS256","typ":"JWT"}`，再把泄露的 RSA 公钥文本当作 HMAC 密钥签名。旧版 PyJWT 会根据令牌中的 `alg` 选择 HS256，并用同一份公钥文本验证，于是攻击者生成的令牌被接受。

PyJWT 新版本会拒绝把 PEM 公钥用于 HMAC；题目固定的 1.5.0 过滤列表只识别 `BEGIN PUBLIC KEY`、`BEGIN CERTIFICATE` 和 `ssh-rsa`，却漏掉 `BEGIN RSA PUBLIC KEY`，因此该格式可以绕过保护。最后把伪造令牌作为 `Authorization: Bearer ...` 访问 `/profile`，并同时带上比赛身份头，即可得到 flag。

`alg=none` 在这里不可行：该实现仍会检查验证密钥与算法组合。漏洞对应的实质是 CVE-2017-11424 所描述的算法/密钥类型混淆，而不是“JWT 天生可随便改 payload”。

## 方法总结

JWT 验证端必须在服务端固定允许的算法，并让密钥类型与算法绑定，不能信任令牌头决定验证方式。调试端点和自动生成的 API 文档也属于攻击面；即使公钥理论上可公开，和旧库的算法混淆组合后仍会变成利用条件。
