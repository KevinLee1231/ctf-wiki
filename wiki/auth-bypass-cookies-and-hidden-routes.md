---
type: family
tags: [web, family, auth, session, access-control, hidden-routes]
skills: [ctf-web]
raw:
  - ../raw/web/auth-bypass-cookies-and-hidden-routes.md
updated: 2026-06-11
---

# Auth Bypass, Cookies and Hidden Routes

## 作用边界

本页是认证与访问控制 family，覆盖 cookie/session、弱签名、隐藏接口、代理 ACL、IDOR、OAuth/open redirect、TLS/HTTP 指纹和信息泄露导致的登录态伪造。它不再作为单个 technique，因为这些案例的利用面不同，但首轮都需要回答同一个问题：服务端到底依据哪个身份字段、路由视图或信任边界做授权。

LLM chatbot jailbreak 只在它直接承担认证或访问控制门禁时保留为本页线索；一般 LLM 绕过应转 [llm-attacks.md](llm-attacks.md)。

## 变体路由

| 信号 | 先验证 | 下一跳 |
|---|---|---|
| cookie 可预测、session 可种植、弱 hash/signature、线性 MAC | token 是否绑定用户、时间、IP、secret、签名算法和服务端状态 | [auth-jwt.md](auth-jwt.md)、[json-duplicate-key-hmac-parser-differential.md](json-duplicate-key-hmac-parser-differential.md) |
| NoSQL/AQL/ORM 查询参与登录 | payload 是否能改变查询结构、返回用户或 role | [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md)、[sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md) |
| hidden admin route、WIP endpoint、IDOR、method bypass | 路由、方法、编码和 ownership check 是否在同一层执行 | [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md) |
| HAProxy/Express/Nginx ACL、`%2F`、大小写、URL 编码 | 代理看到的 path 和应用看到的 path 是否不同 | [path-confusion-to-signed-internal-request-chain.md](path-confusion-to-signed-internal-request-chain.md)、[path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| OAuth email、open redirect、subdomain takeover | 身份提供方、回调地址、email normalization 和域名所有权是否被混用 | [oauth-saml-cors-and-cicd.md](oauth-saml-cors-and-cicd.md) |
| `server-status`、JA4/JA4H、HTTP fingerprint | 信息泄露或指纹匹配是否能构造被信任客户端 | [web-tooling.md](web-tooling.md) |

## 合并与拆分结论

- 保留为 family：它连接多个认证 technique，能提供“先看哪一层授权”的 pivot 价值。
- 不合并进 `auth-edge-cases-and-protocol-bypasses.md`：后者聚焦长尾协议、Unicode、SRP/DH 和数据库表达式边界，是更具体的 technique。
- 不把 LLM 条目继续扩展在本页：LLM 绕过已有 [llm-attacks.md](llm-attacks.md)，本页只记录和访问控制直接相关的交叉信号。

## 常见误判

- 只改 cookie 值，不确认签名、服务端 session store 和用户绑定字段。
- hidden route 扫描命中 200 就认为绕过成功；还要验证数据所有权和敏感动作。
- 代理 ACL 绕过只在本地 curl 成功，没有复现从外部链路到应用层的路径差异。
- OAuth 绕过只看 redirect 参数，没有验证 provider 是否允许未验证邮箱或 subaddressing。

## 关联页面

- [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md)
- [auth-edge-cases-and-protocol-bypasses.md](auth-edge-cases-and-protocol-bypasses.md)
- [auth-jwt.md](auth-jwt.md)
- [json-duplicate-key-hmac-parser-differential.md](json-duplicate-key-hmac-parser-differential.md)
- [oauth-saml-cors-and-cicd.md](oauth-saml-cors-and-cicd.md)
- [llm-attacks.md](llm-attacks.md)

## 原始资料

- [auth-bypass-cookies-and-hidden-routes.md](../raw/web/auth-bypass-cookies-and-hidden-routes.md)
