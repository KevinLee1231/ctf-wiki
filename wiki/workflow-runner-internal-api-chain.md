---
type: technique
tags: [web, pentest, workflow, runner, internal-api, graphql, mcp, build, token, lateral-movement]
skills: [ctf-web, ctf-cloud-infra, ctf-pentest]
raw:
  - ../raw/web/Spirit2026-5-flowforge-wp.md
updated: 2026-07-06
---

# Workflow Runner Internal API Chain

## 适用场景

现代 Web / AI 工作流题把最终 secret 放在内部服务或调试接口后面。攻击者先通过 Web 漏洞提升到 maintainer / builder 级别，再导入自定义 node、processor、plugin 或 build step，让 runner 子进程携带内部 token 请求 localhost 上的 Admin API、GraphQL 或 MCP inspector。

## 识别信号

- 题面是 workflow、CI/CD、AI pipeline、MCP processor、community node 或 build runner。
- 公开界面能看到 workflow index、preview、support incident、artifact ref 或部分源码附件。
- 低权限用户不能直接读 admin secret，但可以导入 node、触发 build 或影响 runner 执行脚本。
- runner 环境变量包含 `RUNNER_TOKEN`、`ADMIN_API_HINT`、`MCP_INSPECTOR_HINT` 等内网访问材料。
- 内部服务使用 GraphQL、debug export ticket、MCP inspector 或 localhost-only API。

## 最小证据

- 已获得足够权限触发 runner 执行自定义脚本或配置。
- 已确认 runner 环境中存在内部 token 或 endpoint hint。
- 已定位内部导出流程，例如 GraphQL `debugExport` 返回 `exportToken`。

## 解法骨架

1. 从公开 workflow、日志、support incident 或附件拿源码和隐藏 endpoint。
2. 利用认证 / parser / preview / webhook 漏洞提升到能导入 node 或触发 build 的角色。
3. 导入最小自定义 node，先打印环境变量和探测 localhost 服务。
4. 在 runner 内请求内部 Admin API；若有 GraphQL，先用源码中的 query 名。
5. 用 runner token 换取一次性 export ticket、bundle token 或 debug handle。
6. 再访问 MCP inspector / debug export endpoint，读取最终 bundle 或 flag。

## 关键变体

- **runner 视角不同于 Web 视角。** Web 会话权限不够，但 runner 进程携带内部信任材料。
- **RCE 只是跳板。** 直接读磁盘可能只拿到 sealed manifest。
- **ticket 两段式导出。** GraphQL 返回一次性 token，MCP inspector 再导出 secret bundle。
- **hint 变量。** 内部 endpoint 常作为环境变量或日志 hint 暴露给 runner。

## 常见陷阱

- 拿到 runner 代码执行后立刻 `cat /flag`，忽略内部授权链。
- 在本机外部请求 `127.0.0.1` 内部接口；必须从 runner 进程内部发起。
- 忽略 preview 或 support incident。
- 手工复制一次性 token 导致过期。

## 关联技巧

- [json-duplicate-key-hmac-parser-differential.md](json-duplicate-key-hmac-parser-differential.md)
- [node-and-prototype.md](node-and-prototype.md)
- [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md)
- [oauth-saml-cors-and-cicd.md](oauth-saml-cors-and-cicd.md)
- [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md)
- [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [HGAME2026-my-little-assistant-wp](../raw/web/HGAME2026-my-little-assistant-wp.md) | MCP 工具只暴露 `py_request`，但 Playwright 禁用 Web 安全后外部页面可请求 localhost MCP JSON-RPC 调用 `py_eval`。 |

## 原始资料

- [Spirit2026-5-flowforge-wp.md](../raw/web/Spirit2026-5-flowforge-wp.md)
