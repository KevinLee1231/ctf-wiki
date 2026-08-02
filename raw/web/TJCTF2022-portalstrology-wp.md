# TJCTF2022 portalstrology

## 题目简述

站点使用 RS256 JWT，但验证时直接信任令牌头中的 `jku`：服务器从任意 URL 下载 JWKS，按 `kid` 取公钥并验证签名。攻击者可以让服务端下载自己的公钥，再用配对私钥签发任意令牌。flag 页面还要求 `iat >= 1652821200`，用户名则必须在内存数据库中存在。

## 解题过程

先正常注册一个账号，确保 `getUserApplied(username)` 不会因不存在的记录报错。随后生成 RSA 密钥对，把公钥发布为可从公网访问的 JWKS，例如：

```json
{
  "keys": [{
    "kty": "RSA",
    "kid": "1",
    "use": "sig",
    "alg": "RS256",
    "n": "...",
    "e": "AQAB"
  }]
}
```

用对应私钥签发 payload，`username` 写刚注册的用户，`iat` 设为门槛时间或更晚；JWT 头设置：

```json
{
  "alg": "RS256",
  "kid": "1",
  "jku": "https://attacker.example/jwks.json"
}
```

把令牌放入名为 `jwt` 的 Cookie 后访问 `/finaid`。服务器请求攻击者的 JWKS，用其中的公钥成功验证攻击者签名，并接受伪造的时间与用户名。页面因此显示 `tjctf{c01l3ges_plz_st0p_th3_l34k5}`。

## 方法总结

非对称签名只保证“令牌由相应私钥签发”，前提是验证者已经可信地确定公钥来源。无约束的 `jku` 把这一信任决定重新交给了攻击者，也附带 SSRF 风险。修复时应固定允许的发行者和 JWKS 地址、校验 `kid` 与声明，并由服务端时间和业务状态决定授权，不能信任客户端自报的 `iat`。
