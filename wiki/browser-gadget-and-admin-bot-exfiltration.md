---
type: technique
tags: [web, xss, admin-bot, dom, exfiltration]
skills: [ctf-web]
raw:
  - ../raw/web/xss-dom-and-browser-tricks.md
  - ../raw/web/RCTF2025-514-wp.md
updated: 2026-07-27
---

# Browser Gadget and Admin-Bot Exfiltration

## 适用场景

受害者/admin bot 在带敏感 cookie、DOM 状态或内网权限的浏览器中访问攻击者可控内容；需要把注入点连接到 DOM gadget、导航、资源请求或 XS-Leak 外带通道。

## 识别信号

- 有 report-to-admin、bot visit、preview 或富文本渲染流程。
- 用户内容进入 HTML/DOM/URL/属性，但 CSP 或 sanitizer 阻止直接脚本。
- 敏感数据仅在受害者浏览器、同源页面或内网 endpoint 可见。

## 最小证据

- 明确 bot 起始 URL、cookie 属性、origin、CSP 和等待时间。
- 在本地浏览器复现注入后的真实 DOM，而非只看源 HTML。
- 自控服务器可记录至少一个由 payload 触发的请求。

## 解法骨架

1. 还原 sanitizer、模板和浏览器解析后的 DOM。
2. 从直接 XSS、DOM clobbering、事件/导航 gadget 到 XS-Leak 依次尝试。
3. 选择 CSP 允许的外带通道，并按 bot 生命周期最小化交互。
4. 在隔离环境复验 cookie/origin 条件，再提交 bot。

## 关键变体

- Stored/reflected/DOM XSS。
- DOM clobbering 或原型/属性 gadget。
- 无脚本 XS-Leak：资源加载、窗口、缓存或导航侧信道。

## 常见陷阱

- 本地测试没有复刻 CSP、SameSite 和 origin。
- Payload 在源字符串中存在，但最终 DOM 被重写。
- 外带 endpoint 使用了 CSP 禁止的 scheme/host。

## 关联技巧

- [xss-dom-and-browser-tricks.md](xss-dom-and-browser-tricks.md)
- [csp-xsleak-and-browser-exfiltration.md](csp-xsleak-and-browser-exfiltration.md)
- [request-view-normalization-differentials.md](request-view-normalization-differentials.md)

## 原始资料

- [xss-dom-and-browser-tricks.md](../raw/web/xss-dom-and-browser-tricks.md)
- [RCTF2025-514-wp](../raw/web/RCTF2025-514-wp.md)
