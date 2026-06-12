---
type: technique
tags: [web, auth-bypass, path-confusion, acl-bypass, cicd, hmac, internal-api]
skills: [ctf-web]
raw:
  - ../raw/web/Spirit2026-5-trust-collapse-wp.md
updated: 2026-05-21
---

# Path Confusion to Signed Internal Request Chain

## 适用场景

Web 题把访问控制、构建系统和内部签名 API 串成一条链。入口不是直接 RCE，而是利用路径解析差异访问管理接口，修改项目配置，把构建流程送到危险实现分支，泄露内部签名密钥，最后伪造合法内部请求读取 flag。

## 识别信号

- 前端入口路径、框架 `PATH_INFO`、自定义路由路径或网关路径之间存在差异。
- `/index.php/admin/...`、`/index.php/internal/...` 等路径在 ACL 看来不是管理路径，但路由仍能命中管理接口。
- 普通用户可以创建项目、仓库、构建任务；管理接口能修改 integration、workspace、channel 等构建参数。
- 最终内部接口要求 HMAC、timestamp、action，而不是依赖浏览器会话。

## 最小证据

- 至少构造出一个前端/网关视角和后端/签名视角不一致的路径样本。
- 能证明签名、白名单或 callback 只覆盖了“看起来安全”的路径，而实际请求落到内部资源。
- 记录规范化前后值：原始 URL、URL decoded、path normalized、代理转发路径和后端收到的路径。

## 解法骨架

1. 固定一条正常 signed request，保存请求体、签名字段、时间戳和目标路径。
2. 构造路径差异：双重编码、`.`/`..`、分号参数、反斜杠、重复 slash、大小写、host/path 混淆。
3. 分别观察 gateway、signer、backend 的响应差异，优先找状态码、错误信息和日志回显。
4. 让签名视角保持允许路径，让后端实际访问 internal endpoint、metadata、admin API 或 workflow runner。
5. 若进入内部 API，再按认证、SSRF、Node require/RCE 或文件读取继续 pivot。

## 关键变体

| 变体 | 复用重点 |
|---|---|
| Double URL encoding | 签名层只解一次，后端或代理再解一次。 |
| Path normalization | `/safe/../admin` 在不同层处理顺序不同。 |
| Signed callback | HMAC 保护的是 callback 参数文本，不一定保护最终请求目的地。 |
| Internal request chain | 第一次差异通常只给 SSRF/内部访问，后面还要串权限或 RCE。 |

## 常见陷阱

- 只验证浏览器可访问路径，没有确认后端实际收到什么。
- 自动 URL 库会规范化 payload，导致手工构造的差异被客户端消掉。
- HMAC 错误就放弃；应先判断签名覆盖的到底是哪个字段和哪个编码层。
- 把路径混淆当成普通 path traversal，忽略签名/代理/后端三视角差异。

## 关联技巧

- [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md)
- [oauth-saml-cors-and-cicd.md](oauth-saml-cors-and-cicd.md)
- [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md)
- [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md)

## 原始资料

- [Spirit2026-5-trust-collapse-wp.md](../raw/web/Spirit2026-5-trust-collapse-wp.md)
