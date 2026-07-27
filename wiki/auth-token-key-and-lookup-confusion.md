---
type: technique
tags: [web, jwt, auth, key-confusion, kid, jku]
skills: [ctf-web]
raw:
  - ../raw/web/auth-jwt.md
updated: 2026-07-27
---

# Auth Token Key and Lookup Confusion

## 适用场景

JWT/JWE/签名 token 的算法、密钥类型或 key lookup 由攻击者可控 header/claim 影响，使验证端选错算法、把公钥当共享密钥，或从可控位置加载 key。

## 识别信号

- Token header 含 `alg/kid/jku/x5u/jwk`，服务端据此选择验证路径。
- 同一 endpoint 接受多种算法/密钥类型，或公开 JWKS/公钥。
- `kid` 进入文件、SQL、字典或缓存查找。

## 最小证据

- 解码原 token，确认实际算法、key id 和 claim 约束。
- 证明验证库/应用允许攻击者影响算法或 key source。
- 伪造 token 必须通过服务端验证并获得原本无权访问的状态。

## 解法骨架

1. 还原 token 解析、算法白名单和 key resolution 顺序。
2. 分别测试 `none`、对称/非对称混淆和 header key injection。
3. 对 `kid/jku` 构造最小受控 key source，并保持 claim 结构合法。
4. 重新签名后验证 issuer、audience、expiry 和业务权限。

## 关键变体

- Algorithm confusion：验证端根据 token 自报算法切换。
- Key-type confusion：RSA/ECDSA 公钥被当作 HMAC secret。
- Key lookup injection：`kid/jku/x5u` 指向文件、数据库或攻击者 JWKS。

## 常见陷阱

- 只改 payload 不重新计算签名。
- 库已固定算法，却仍机械尝试 `alg=none`。
- 伪造 token 签名通过，但 claim 业务校验失败。

## 关联技巧

- [auth-jwt.md](auth-jwt.md)
- [session-and-access-control-state-confusion.md](session-and-access-control-state-confusion.md)
- [json-duplicate-key-hmac-parser-differential.md](json-duplicate-key-hmac-parser-differential.md)

## 原始资料

- [auth-jwt.md](../raw/web/auth-jwt.md)
