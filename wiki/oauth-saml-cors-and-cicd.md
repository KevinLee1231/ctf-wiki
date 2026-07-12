---
type: family
tags: [web, family, oauth, oidc, saml, cors, cicd]
skills: [ctf-web, ctf-cloud-infra]
raw:
  - ../raw/web/oauth-saml-cors-and-cicd.md
updated: 2026-06-12
---

# OAuth, SAML, CORS and CI/CD

## 作用边界

本页是身份联邦、跨域信任和 CI/CD 凭据链 family。它覆盖 OAuth/OIDC、SAML、CORS、Git 历史泄露、CI/CD 变量、身份提供方 API、Guacamole/TeamCity 等平台化登录与部署链。

共同问题是：应用把身份、redirect、Origin、token、仓库或 runner 变量托付给外部系统。首轮要确认“哪个系统被信任，以及信任条件是否可伪造”。

边界在于“跨系统信任”：如果只是在应用本地 cookie/session/hidden route/IDOR/代理 ACL 层绕过授权，回到 [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md)；如果 token 本体的算法、key lookup 或 claim 可伪造，转 [auth-jwt.md](auth-jwt.md)。

## 识别信号

- 登录流程出现 OAuth/OIDC/SAML provider、`redirect_uri`、`state`、`code`、`id_token`、ACS URL 或 metadata。
- CORS 响应反射 Origin，或凭据请求允许跨站读取敏感接口。
- `.git`、CI 配置、pipeline log、runner、artifact、TeamCity/Jenkins/GitLab 等暴露 token 或变量。
- 内部身份 API、Guacamole connection、SSO 自动化或登录页被修改后能影响管理员。

## 最小证据

- 完整记录一次正常认证流：请求顺序、回调参数、cookie、token、issuer/audience、state 和 redirect。
- 能验证一个信任边界被绕过：redirect 域名、state 绑定、email verification、Origin、签名、CI 变量作用域。
- CI/CD 题要确认 token 权限和执行位置：只能读 artifact，还是能改 pipeline、调用内部 API 或执行命令。
- CORS 题要证明浏览器侧可带凭据读取敏感响应，不只看响应头。

## 路由表

| 证据 | 先验证 | 下一跳 |
|---|---|---|
| OAuth open redirect / token theft | `redirect_uri` 是否严格匹配，code/token 是否泄露到攻击者域 | [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md) |
| OIDC `id_token` 可改 | issuer、audience、nonce、email_verified 和签名是否校验 | [auth-jwt.md](auth-jwt.md) |
| `state` 缺失或不绑定 session | 是否能把受害者 code 绑定到攻击者 session | [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md) |
| SAML/SSO 自动化 | ACS、NameID、签名覆盖范围和 XML canonicalization 是否正确 | [xml-command-and-graphql-injection.md](xml-command-and-graphql-injection.md) |
| CORS 反射 Origin | 是否允许 credentials，敏感接口是否可被浏览器读取 | [csp-xsleak-and-browser-exfiltration.md](csp-xsleak-and-browser-exfiltration.md) |
| Git 历史或 CI 变量泄露 | token 是否仍有效、权限范围和环境隔离是否足够 | [workflow-runner-internal-api-chain.md](workflow-runner-internal-api-chain.md) |
| TeamCity/Guacamole/IdP API | 凭据能否从管理 API 扩展到连接参数或 RCE | [known-cves-and-n-day-exploits.md](known-cves-and-n-day-exploits.md) |
| 登录页投毒 | 管理员是否会访问被修改页面，凭据是否回传 | [xss-dom-and-browser-tricks.md](xss-dom-and-browser-tricks.md) 或 [workflow-runner-internal-api-chain.md](workflow-runner-internal-api-chain.md) |

## 合并与拆分结论

- 保留为 family：OAuth、SAML、CORS 和 CI/CD 的 sink 不同，但都围绕跨系统信任边界。
- 不合并进 `auth-bypass-cookies-and-hidden-routes.md`：认证总入口只决定进入本页，本页负责联邦身份和平台凭据链的细分。
- 暂不拆 OAuth/OIDC/SAML/CORS 小页：当前 raw 更偏平台链路，拆分会削弱从 token 到 CI/API 的连续转向。

## 常见误判

- 只看 redirect 成功，没有验证 code/token 是否真正进入攻击者控制。
- CORS 只看 `Access-Control-Allow-Origin`，没验证 credentials 和浏览器可读性。
- CI token 泄露后不查权限范围，导致停在信息泄露而不是 runner/API 接管。
- SAML/OIDC 只解码 token，不检查 issuer、audience、nonce 和签名覆盖。

## 关联页面

- [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md)
- [auth-jwt.md](auth-jwt.md)
- [workflow-runner-internal-api-chain.md](workflow-runner-internal-api-chain.md)
- [known-cves-and-n-day-exploits.md](known-cves-and-n-day-exploits.md)
- [csp-xsleak-and-browser-exfiltration.md](csp-xsleak-and-browser-exfiltration.md)
- [web-tooling.md](web-tooling.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [RCTF2025-auth-wp](../raw/web/RCTF2025-auth-wp.md) | Node IdP 中 `parseInt(false)` 与 MySQL `TINYINT` 存储语义不一致可注册 admin，再利用 SP XML Signature Wrapping 让业务读取未签名 Assertion。 |

## 原始资料

- [oauth-saml-cors-and-cicd.md](../raw/web/oauth-saml-cors-and-cicd.md)
