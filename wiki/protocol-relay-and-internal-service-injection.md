---
type: technique
tags: [web, protocol, ssrf, redis, internal-service]
skills: [ctf-web, ctf-pentest]
raw:
  - ../raw/web/WMCTF2025-rustdesk-change-client-backend-wp.md
updated: 2026-06-12
---

# Protocol Relay and Internal Service Injection

## 适用场景

第三方客户端、远控、消息转发、代理或 rendezvous 协议允许攻击者控制下游连接目标、relay 地址、peer id、uuid、metadata 等字段。目标主机主动连向攻击者指定的内网地址或本机服务，且协议字段会以可预测明文进入 TCP 流，从而把“协议转发”转成内网服务命令注入。

## 识别信号

- 组件分成控制面和数据面，例如 rendezvous server、relay server、client backend、agent callback。
- 只有部分消息路径校验 key、token 或公钥，另一些 relay/request 消息只做转发。
- 可控字段会被编码成 protobuf、JSON、line protocol 或自定义字符串，但最终仍包含可控明文。
- 目标本机存在 Redis、Docker API、数据库、RPC 或未鉴权内部管理服务。

## 最小证据

- 能证明目标客户端会按攻击者提供的 relay/internal 地址主动连接。
- 能抓到目标发往内部服务的第一段数据，并确认可控字段在其中的字节位置。
- 内部服务协议允许 CRLF、批量字符串、gopher-like body 或其它可拼接命令的明文注入。

## 解法骨架

1. 审计控制面消息，比较哪些分支校验 key，哪些分支只按 peer id/relay 地址转发。
2. 构造最小消息，让目标连接 `127.0.0.1:<port>` 或内网服务，先用可观测端口确认连接发生。
3. 把可控字段改成内部协议命令，验证命令边界、编码层和禁止字符。
4. 如果目标是 Redis，优先判断是否能写配置、主从同步加载模块、写 crontab/authorized_keys 或利用已知版本链。
5. 完成 RCE 后再处理远控软件自身的 view-only、文件传输、剪贴板或终端限制。

## 关键变体

| 变体 | 复用重点 |
|---|---|
| Relay 地址可控 | 攻击不是直接访问内网，而是让目标客户端替你访问本机或内网服务。 |
| 字段明文穿透 | protobuf/JSON 包装不等于安全；只要字符串字段原样进入 TCP 流，就能拼协议。 |
| Key 校验路径不一致 | 先比较同一组件中 PunchHole、RequestRelay、callback、heartbeat 等消息分支。 |
| 内网服务二段利用 | Redis、Docker API、数据库等通常还要准备 rogue server、模块、payload 文件或二段命令。 |

## 常见陷阱

- 只测试官方客户端连接流程，忽略可以手写控制面消息。
- 以为主控端拿不到服务器 key 就不能利用；真正入口可能是目标受控端主动连接。
- 可控字段里出现单引号、空格或二进制长度字段时，payload 需要按协议重新编码，而不是直接复制 Redis/gopher payload。
- Redis 主从同步或模块加载依赖版本、配置和网络可达性，命令注入成功不等于立即 RCE。

## 关联技巧

- [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md)
- [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md)
- [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md)
- [network-covert-auth-and-reassembly.md](network-covert-auth-and-reassembly.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [WMCTF2025-rustdesk-change-client-backend-wp](../raw/web/WMCTF2025-rustdesk-change-client-backend-wp.md) | RustDesk hbbs 只在部分消息路径校验 key，伪造 `RequestRelay` 让受控端连接 `127.0.0.1:6379`，再用 uuid 中的 CRLF Redis 命令触发主从同步加载模块 RCE。 |

## 原始资料

- [WMCTF2025-rustdesk-change-client-backend-wp](../raw/web/WMCTF2025-rustdesk-change-client-backend-wp.md)
