---
type: technique
tags: [web, technique, csp, xs-leak, browser-exfiltration]
skills: [ctf-web]
raw:
  - ../raw/web/csp-xsleak-and-browser-exfiltration.md
updated: 2026-06-11
---

# CSP, XS-Leak and Browser Exfiltration

## 适用场景

已经确认攻击发生在受害者浏览器或 admin bot 中，但直接读 secret、直接执行外部脚本或直接回传数据被 CSP、同源策略、过滤器、MIME、nonce 或跨站限制拦住。此时核心技巧不是“让 XSS 弹窗”，而是构造一个可重复的浏览器 oracle，把 secret 的位、字符、长度或状态逐步外带。

## 识别信号

- 有 admin bot、report URL、受害者 cookie、magic link、跨站跳转或前端-only secret。
- CSP 限制脚本来源，但允许特定 CDN、`base`、`prefetch`、图片、字体、CSS、JSONP 或 cloud function 域。
- 页面包含可控 CSS/HTML/URL/hash/postMessage/iframe，但不能直接读取响应 body。
- 目标状态能影响资源加载、尺寸、字体、时间、错误、redirect、缓存或 postMessage 返回。

## 最小证据

- 能让受害者浏览器访问攻击者控制页面或注入片段。
- 能证明一个候选条件会改变可观测行为，例如图片是否加载、字体宽度、请求是否发出、postMessage 返回、时间差或 cache 命中。
- 能把单次 oracle 自动化成多轮提取，而不是只得到一个偶然响应。

## 解法骨架

1. 固定受害者上下文：bot 是否带 cookie、CSP 具体策略、可控 sink、可用出站通道。
2. 选择 oracle：CSS selector/font/container query、image timing、prefetch、JSONP/XSSI、postMessage、redirect 或缓存状态。
3. 设计候选集：先提长度/前缀/字符集，减少每轮分支。
4. 让每轮 oracle 只回答一个小问题，并把命中结果回传到攻击者服务。
5. 用已恢复前缀继续构造下一轮 payload，直到 forward check 能访问目标资源或复现 secret。

## 浏览器外带分支

| 外带条件 | 判断重点 |
|---|---|
| CSP 白名单 CDN/Cloud Function 绕过 | 关键不是找到任意第三方域，而是找到能执行 attacker-controlled JS 或能回传数据的白名单域。 |
| `base` tag / nonce / prefetch 绕过 | 先确认浏览器实际解析顺序；nonce 保护脚本不等于保护所有资源加载。 |
| CSS font glyph / container query 外带 | secret 字符必须能影响布局、宽度或匹配选择器；每个候选要产生可观测网络请求。 |
| XS-Leak timing / image load | 需要多次采样和噪声控制；最好让成功/失败差异足够大。 |
| JSONP/XSSI callback 外带 | callback 或 MIME 绕过只解决执行问题，还要确认敏感数据在响应中且 cookie 会被带上。 |
| postMessage oracle | 检查 origin 校验、`data:`/null origin、iframe sandbox 和消息格式。 |

## 常见陷阱

- payload 在攻击者浏览器中可执行，但 admin bot 的 CSP、cookie、浏览器版本或导航规则不同。
- 只证明 CSP 可以绕过，没有证明 secret 能外带。
- timing oracle 没有重复采样，误把网络抖动当成字符命中。
- JSONP/XSSI 忽略 SameSite cookie、CORS、MIME sniffing 和 callback 过滤。

## 关联技巧

- [xss-dom-and-browser-tricks.md](xss-dom-and-browser-tricks.md)
- [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md)
- [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md)
- [polyglot-url-tricks-and-ssrf-leaks.md](polyglot-url-tricks-and-ssrf-leaks.md)
- [web-tooling.md](web-tooling.md)

## 原始资料

- [csp-xsleak-and-browser-exfiltration.md](../raw/web/csp-xsleak-and-browser-exfiltration.md)
