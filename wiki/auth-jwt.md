---
type: technique
tags: [web, technique]
skills: [ctf-web]
raw:
  - ../raw/web/auth-jwt.md
updated: 2026-05-21
---

# JWT & JWE Token Attacks

## 适用场景

HTTP 行为、认证授权、服务端解析差异、文件/路径处理、模板或浏览器执行模型是主要障碍。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 请求、路由、Cookie、Header、body 或服务端状态出现可控差异。
- 源码或响应能定位到中间件、鉴权、解析器、模板、上传或内部 API。
- 能用最小 HTTP payload 复现行为变化。
- 题面或 raw 线索能落到这些关键词之一：Algorithm None、Algorithm Confusion (RS256 to HS256)、Weak Secret Brute-Force、Unverified Signature (Crypto-Cat)、JWK Header Injection (Crypto-Cat)、JKU Header Injection (Crypto-Cat)、KID Path Traversal (Crypto-Cat)、JWT Balance Replay (MetaShop Pattern)。

## 最小证据

- 已完成主方向判断，并确认本页技巧比相邻技巧更能解释当前证据。
- 至少有一个可复现输入、输出、文件结构、数学关系、协议行为或运行时状态。
- 能指出 raw 案例中哪一个变体与当前题最接近，以及不同点在哪里。

## 解法骨架

1. 固定登录态和正常业务流。
2. 构造最小请求验证一个主假设。
3. 把单点漏洞串成 secret/token/internal API/RCE。
4. 写可重放 solver 并记录关键响应。

## 关键变体

| 变体 | 复用重点 |
|---|---|
| Algorithm None | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Algorithm Confusion (RS256 to HS256) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Weak Secret Brute-Force | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Unverified Signature (Crypto-Cat) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| JWK Header Injection (Crypto-Cat) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| JKU Header Injection (Crypto-Cat) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| KID Path Traversal (Crypto-Cat) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| JWT Balance Replay (MetaShop Pattern) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| JWE Token Forgery with Exposed Public Key (UTCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| AES Cookie Length-Field Truncation + CRC32 Swap (DefCamp 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| JWT 速查 | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

## 常见陷阱

- 只按关键词跳页，没有先构造最小证据。
- 照搬 raw 中的一次性 payload，没有检查当前题的边界条件。
- 忽略相邻技巧之间的 pivot，导致在错误方向上继续投入时间。

## 关联技巧

- [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md)
- [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md)
- [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md)
- [csp-xsleak-and-browser-exfiltration.md](csp-xsleak-and-browser-exfiltration.md)
- [json-duplicate-key-hmac-parser-differential.md](json-duplicate-key-hmac-parser-differential.md)

## 原始资料

- [auth-jwt.md](../raw/web/auth-jwt.md)
