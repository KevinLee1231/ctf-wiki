---
type: family
tags: [web, family, auth, jwt, jwe, token]
skills: [ctf-web]
raw:
  - ../raw/web/auth-jwt.md
updated: 2026-06-12
---

# JWT and JWE Token Attacks

## 作用边界

本页是 JWT/JWE 与签名 cookie 的 token family。它覆盖 `alg=none`、RS256/HS256 混淆、弱 secret、未校验签名、JWK/JKU/KID 注入、JWE 公钥误用、余额/权限 replay，以及少量同类签名 cookie 的长度字段或 CRC 绕过。

它不替代认证总入口。只有当身份或权限主要由可重放 token、header 参数、签名算法或密钥来源决定时才进入本页。

## 识别信号

- Cookie、Authorization header 或 localStorage 中存在三段 JWT、JWE、base64 JSON token 或自定义签名 cookie。
- token header 包含 `alg`、`kid`、`jwk`、`jku`、`x5u`、`enc`、`zip` 等可控字段。
- 服务端把公钥、文件路径、远程 JWK、弱 secret、环境变量或用户字段当作验签依据。
- 修改 claim 后响应变化，或签名失败/成功状态可被区分。

## 最小证据

- 正常 token 的 header、payload、签名算法、关键 claim 和过期时间。
- 验签失败和权限失败的响应差异，避免把无效 token 当成无权限。
- secret/key 来源线索：源码、`.env`、JWK URL、`kid` 文件路径、公钥、默认 secret 或弱字典。
- 成功伪造后能触发一个敏感动作：admin claim、余额、用户 ID、文件读或内部 API。

## 路由表

| 证据 | 先验证 | 下一跳 |
|---|---|---|
| `alg=none` | 服务端是否接受 unsigned token，库版本是否允许 none | 直接构造 admin claim |
| RS256 -> HS256 | 公钥是否被当 HMAC secret 使用 | 用服务端公钥重签 HS token |
| 弱 secret | secret 短、默认、来自源码或环境泄露 | 字典/规则爆破后重签 |
| 未校验签名 | 修改 payload 后签名不变仍生效 | 确认仅 decode 未 verify |
| JWK/JKU/X5U 注入 | 服务端会拉取 attacker-controlled key | 控制 key set 并重签 |
| `kid` 注入/路径穿越 | `kid` 参与本地文件或 key lookup | 读 `/dev/null`、公开文件或路径穿越到已知 key |
| JWE 公钥误用 | 加密 token 可由公开材料伪造或降级 | 确认 JWE header、key management 和解密失败差异 |
| 签名 cookie 长度/CRC | token 不是 JWT 但结构可控、MAC 弱或字段可交换 | 转 [block-mode-misuse-family.md](block-mode-misuse-family.md) 或 [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md) |
| JSON duplicate key / parser 差异 | 验签和业务读取不同字段 | 转 [json-duplicate-key-hmac-parser-differential.md](json-duplicate-key-hmac-parser-differential.md) |

## 合并与拆分结论

- 保留为 family：JWT/JWE 的攻击点分散在算法、key lookup、远程 key、claim 和解析差异，适合作为 token 二级路由。
- 不合并进 `auth-bypass-cookies-and-hidden-routes.md`：认证入口页只决定是否进入 token 路线。
- 不拆 `jwt-alg-confusion.md` 等小页：当前 raw 还不足以支撑多个独立 technique 页。

## 常见误判

- 解码成功不等于绕过，必须确认服务端验签和业务授权都通过。
- RS/HS 混淆要求服务端确实把公钥当 HMAC secret，不能只改 header。
- `kid` 路径 payload 要考虑拼接后缀、缓存和 key store 格式。
- JWK/JKU 需要确认服务端会出网并信任远程 key，不是所有库都拉取。

## 关联页面

- [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md)
- [oauth-saml-cors-and-cicd.md](oauth-saml-cors-and-cicd.md)
- [json-duplicate-key-hmac-parser-differential.md](json-duplicate-key-hmac-parser-differential.md)
- [block-mode-misuse-family.md](block-mode-misuse-family.md)
- [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md)
- [web-tooling.md](web-tooling.md)

## 原始资料

- [auth-jwt.md](../raw/web/auth-jwt.md)
