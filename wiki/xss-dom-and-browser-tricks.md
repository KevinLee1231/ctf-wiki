---
type: family
tags: [web, family, xss, dom, browser, exfiltration]
skills: [ctf-web]
raw:
  - ../raw/web/xss-dom-and-browser-tricks.md
updated: 2026-06-11
---

# XSS, DOM and Browser Tricks

## 作用边界

本页是浏览器执行与外带 family。共同模式是攻击者需要让浏览器、admin bot、DOM sink、缓存、CSP、MIME、Shadow DOM 或前端框架在受害者上下文中执行或泄露数据。它不再作为单一 technique，因为 XSS payload、DOM clobbering、cache poisoning、XS-Leak 和 admin bot 绕过的证据结构不同。

## 变体路由

| 信号 | 先确认 | 下一跳 |
|---|---|---|
| 反射/存储 XSS、过滤 `<script>`、Unicode/hex 绕过 | payload 是否进入 HTML/属性/JS/URL/CSS 哪个上下文 | [csp-xsleak-and-browser-exfiltration.md](csp-xsleak-and-browser-exfiltration.md) |
| DOMPurify、DOM clobbering、Shadow DOM、jQuery hashchange | sink、source、sanitize 后结构和浏览器解析结果是否一致 | [csp-xsleak-and-browser-exfiltration.md](csp-xsleak-and-browser-exfiltration.md) |
| admin bot、`javascript:` URL、magic link、redirect chain | bot 是否带 cookie，是否点击/访问攻击者可控 URL，CSP 和跳转链如何限制 | [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md) |
| cache poisoning、Content-Type/MIME mismatch、JPEG+HTML polyglot | 缓存 key、响应头、扩展名和浏览器 MIME sniffing 是否错位 | [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md) |
| CSS font/container query、image timing、postMessage、JSONP/XSSI | 是否能构造稳定 oracle，逐步外带 secret 或状态位 | [csp-xsleak-and-browser-exfiltration.md](csp-xsleak-and-browser-exfiltration.md) |
| JSFuck、AngularJS sandbox、React internal state | 前端框架/编码技巧是否只是到达 sink 的手段，还是题目主障碍 | [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) 或本页 raw |

## 合并与拆分结论

- 保留为 family：它负责把浏览器侧题目分到 XSS、DOM、CSP/XS-Leak、缓存/MIME、admin bot 和框架绕过。
- 不与 [csp-xsleak-and-browser-exfiltration.md](csp-xsleak-and-browser-exfiltration.md) 合并：后者是具体 technique，重点是受 CSP/同源/无直接读权限限制时如何外带数据。
- 不单独拆 JSFuck 页面：当前它更像编码/到达 sink 的辅助技巧，优先从本页或 [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) 查询。

## 常见误判

- 只看到 XSS 字符串执行，没有确认是否在 admin/bot 的认证上下文中执行。
- CSP 绕过只验证脚本加载，不验证能否把 secret 外带到攻击者控制端。
- DOM 题只看源码 grep，没有在真实浏览器里观察 sanitizer、URL parser 和事件触发时机。
- 缓存题忽略 cache key，导致本地复现成功但污染不到受害者路径。

## 关联页面

- [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md)
- [csp-xsleak-and-browser-exfiltration.md](csp-xsleak-and-browser-exfiltration.md)
- [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md)
- [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md)
- [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md)

## 原始资料

- [xss-dom-and-browser-tricks.md](../raw/web/xss-dom-and-browser-tricks.md)
