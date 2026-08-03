---
type: technique
tags: [reverse, web, malware, browser-extension, manifest, service-worker, messaging]
skills: [ctf-reverse, ctf-web, ctf-malware]
raw:
  - ../raw/web/N1CTF2022-do-not-touch-my-localhost-wp.md
updated: 2026-08-03
---

# Browser Extension Manifest, Worker and Message Flow

## 适用场景

对象是 CRX/XPI/解包扩展，关键逻辑分散在 `manifest.json`、background page/service worker、content script、page context、popup/options、storage、declarative rules 或 native messaging 中。目标是建立“权限—执行上下文—消息—状态—source/sink”图。

## 识别信号

- `permissions`、`host_permissions`、`content_scripts`、`background.service_worker`、`externally_connectable`、`web_accessible_resources` 或 `nativeMessaging`。
- Manifest V2 background page 与 Manifest V3 service worker 的生命周期差异导致静态逻辑相同、运行状态不同。
- `runtime.sendMessage/onMessage`、port/connect、`tabs.sendMessage`、`window.postMessage`、storage change 或 DOM event 组成跨上下文链。
- `webRequest`、declarativeNetRequest、cookies、downloads、clipboard、history、proxy 或 native host 成为高影响 sink。

## 最小证据

- 保留原始归档和解包结果哈希，记录扩展 ID/版本、manifest version、入口脚本与权限。
- 对每个消息记录 sender context、receiver context、message schema、返回值和进入的 sink。
- 用一个可控页面和一个可重现事件验证 source-to-sink 路径，不仅凭权限表推测行为。

## 解法骨架

1. 解包后从 `manifest.json` 建立入口和权限表，区分显式权限、host 范围和只在特定 URL 注入的 content script。
2. 按执行上下文绘图：background/service worker、content script isolated world、page world、extension page、offscreen document、native host。
3. 搜索 message listener/sender、storage key、network API 和高影响 sink，为消息 schema 建立表，不只搜 URL 和 token。
4. 沿用户输入、页面 DOM、网络响应或 external message 向前追到 sink；同时向后追踪高权限 sink 的全部 caller。
5. 动态验证时使用专用临时浏览器 profile、可控测试页面和受控网络；将 worker 唤醒/终止与持久状态分开记录。
6. 把结果写成“可达 source → 信任检查 → 消息边界 → 高影响 sink”，并用一个实际事件复现。

## 路由边界

| 决定性主障碍 | 归属 |
|---|---|
| 扩展代码、消息图和运行行为还原 | `ctf-reverse` |
| 扩展与站点的认证、DOM、同源、CSP 或 Web 业务边界 | `ctf-web` |
| 窃密、持久化、C2、恶意更新或 native host 载荷 | `ctf-malware` |

## 常见陷阱

- 把声明权限当成已发生行为，没有验证代码路径和触发条件。
- 忽略 content script 与 page world 隔离，把 `window.postMessage` 和 extension runtime message 混为一层。
- 在日常浏览器 profile 加载不可信扩展，导致真实 cookie、token 或历史数据暴露。
- 忽略 MV3 worker 会休眠，把生命周期导致的状态丢失误判为分析结论。

## 关联技巧

- [browser-javascript-runtime-reconstruction.md](browser-javascript-runtime-reconstruction.md)
- [scripts-and-obfuscation.md](scripts-and-obfuscation.md)
- [xss-dom-and-browser-tricks.md](xss-dom-and-browser-tricks.md)
- [malware-c2-session-key-and-protocol-recovery.md](malware-c2-session-key-and-protocol-recovery.md)

## 原始资料

- [N1CTF2022-do-not-touch-my-localhost-wp.md](../raw/web/N1CTF2022-do-not-touch-my-localhost-wp.md)
