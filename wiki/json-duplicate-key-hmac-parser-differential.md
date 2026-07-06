---
type: technique
tags: [web, json, hmac, parser-differential, duplicate-key, webhook, auth-bypass]
skills: [ctf-web, ctf-crypto]
raw:
  - ../raw/web/Spirit2026-5-flowforge-wp.md
updated: 2026-05-21
---

# JSON Duplicate Key HMAC Parser Differential

## 适用场景

Webhook、callback 或 internal API 使用请求体 HMAC 校验完整性，但签名逻辑和业务逻辑使用不同 JSON 解析语义。攻击者构造重复 key，使签名视角看到安全事件，业务视角看到危险事件，从而绕过授权或角色限制。

## 识别信号

- 源码中存在 `firstKeyJsonParse`、stream parser、regex parser，和标准 `JSON.parse` 同时出现。
- HMAC 计算基于“规范化后的对象”而不是原始字节串。
- 业务字段名是 `event`、`action`、`role`、`target` 这类少数关键 key。
- 当前低权限身份只能签 benign event，但业务端支持 promote、admin、export 等高权限 event。

## 最小证据

- 已确认签名校验和业务逻辑都解析同一 JSON body，但使用的解析器、canonicalization 或重复 key 处理规则不同。
- 能构造一个重复 key 样本，使签名视角看到安全值，业务视角看到危险值。
- 能解释服务端采用 first-wins、last-wins、merge、streaming parser 或 framework body parser 中的哪一种语义。

## 解法骨架

1. 抓正常请求，固定签名字段、body 原文、时间戳和 canonicalization 规则。
2. 用原始字节手写 JSON，不要用会去重的 JSON 库重序列化。
3. 构造 `{"role":"user","role":"admin"}` 或同类重复 key，测试响应差异。
4. 分别验证 HMAC 是否覆盖原文、排序后字段、部分字段或解析后的对象。
5. 一旦业务视角切到高权限，将其串到 internal API、workflow runner、SSRF 或 RCE。

## 解析差异分支

| 差异形态 | 利用判断 |
|---|---|
| first-wins vs last-wins | 签名层和业务层对重复 key 取值不同。 |
| raw body HMAC | HMAC 覆盖字节原文时，payload 不能被客户端库改写。 |
| canonical JSON | 排序、空格、转义和 Unicode 规范化都会影响签名。 |
| parser differential chain | 该漏洞常只是进入内部 API 的第一跳。 |

## 常见陷阱

- 用普通 JSON 工具重写 payload，导致重复 key 被自动丢弃。
- HMAC canonicalization 与服务端不一致。
- 只构造 dangerous body，没有让签名视角满足低权限白名单。
- 把该问题误判为纯 crypto；真正漏洞通常在 Web parser 语义差异。

## 关联技巧

- [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md)
- [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md)
- [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md)
- [workflow-runner-internal-api-chain.md](workflow-runner-internal-api-chain.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [ACTF2026-12307-wp](../raw/web/ACTF2026-12307-wp.md) | SSO continuation 只做字符串包含建立 settlement 信任，随后 ORDER BY 盲注泄露 `claim_salt`，最终用 duplicate-key print plan 混淆进入私有打印通道。 |
| [NCTF2026-n-rustpica-wp](../raw/web/NCTF2026-n-rustpica-wp.md) | 静态调试目录泄露后台凭据后，Rust/Serde `untagged enum` 和未知字段忽略导致工作流状态绕过。 |

## 原始资料

- [Spirit2026-5-flowforge-wp.md](../raw/web/Spirit2026-5-flowforge-wp.md)
