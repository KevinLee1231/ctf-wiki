---
type: family
tags: [web, family, auth, jwt, jwe, token]
skills: [ctf-web]
raw:
  - ../raw/web/auth-jwt.md
updated: 2026-07-27
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

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| `alg/kid/jku/x5u/jwk` 影响算法、密钥类型或 key source | [auth-token-key-and-lookup-confusion.md](auth-token-key-and-lookup-confusion.md) |
| Cookie/header/session/代理与业务层读取不同身份状态 | [session-and-access-control-state-confusion.md](session-and-access-control-state-confusion.md) |
| JSON 重复键或验签视图与业务解析视图不一致 | [json-duplicate-key-hmac-parser-differential.md](json-duplicate-key-hmac-parser-differential.md) |

## 合并与拆分结论

- 保留为 family：JWT/JWE 的攻击点分散在算法、key lookup、远程 key、claim 和解析差异，适合作为 token 二级路由。
- 不合并进 `auth-bypass-cookies-and-hidden-routes.md`：认证入口页只决定是否进入 token 路线。
- 算法/key lookup 混淆、session 状态差异和 JSON 验签视图差异已有独立 technique；本页保留 JWT/JWE 变体分流。

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

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [ACTF2026-aaa26-wp](../raw/web/ACTF2026-aaa26-wp.md) | Mongo `$regex` 盲注恢复 reviewer invite code，vm2 里用 Buffer slab 泄露 JWT secret，伪造 admin 后上传伪 PDF/SVG 让 ImageMagick `text:/flag` 渲染。 |
| [D3CTF2025-d3invitation-wp](../raw/cloud-infra/D3CTF2025-d3invitation-wp.md) | MinIO STS session token 是 JWT，object_name 可注入 policy 影响对象访问权限。 |
| [0xGame2022-week4-profile-wp](../raw/web/0xGame2022-week4-profile-wp.md) | 漏洞本质是认证状态与业务对象生命周期脱节：JWT 在用户删除后仍然有效，而查询不存在的用户又返回了一个缺少权限字段的空对象，最终形成 fail-open。审计类似逻辑时，应同时检查 token 撤销、对象不存在时的返回值，以及权限字段缺失时是否默认拒绝。 |
| [0xGame2025-week3-长夜月-wp](../raw/web/0xGame2025-week3-长夜月-wp.md) | 本题把未验签 JWT 与原型链污染串联起来。前者说明“能解码”不等于“已认证”，安全鉴权必须调用 `jwt.verify()` 并限制算法；后者说明递归合并不可信 JSON 时，应拒绝 `__proto__`、`prototype`、`constructor` 等特殊键。这里被污染的是共享的 `CONFIG` 原型对象，而不是全局 `Object.prototype`，准确追踪对象的原型关系才能看清 flag 条件为何成立。 |
| [ACTF2023-easylatex-wp](../raw/web/ACTF2023-easylatex-wp.md) | 本题串联了身份、浏览器和服务端请求三个信任边界。JWT 验证必须固定允许的算法，并禁止把非对称公钥当作 HMAC 密钥；允许用户控制资源根目录时，CSP 中的可信 CDN 也会变成脚本托管点；服务端请求更不应根据身份字段选择绝对 URL，也不能无条件转发浏览器 Cookie。HttpOnly 只阻止 JavaScript 直接读取 Cookie，无法阻止应用自己的接口把 Cookie 复制到攻击者可控请求中。 |
| [D3CTF2024-Doctor-wp](../raw/web/D3CTF2024-Doctor-wp.md) | 可用链路是“伪 WebSocket 头绕过 JWT→无效 `source_id` 留下零值结构→`DataBase` 污染完整 DSN→`allowAllFiles=true`→Rogue MySQL 的 LOCAL INFILE 文件读取”。最隐蔽的环节不是单次拼接，而是同一字符串先经过 `FormatDSN`、再经过按最后一个 `/` 切分的 `ParseDSN`，两段看似合理的逻辑组合出了参数注入。 |

## 原始资料

- [auth-jwt.md](../raw/web/auth-jwt.md)
