---
type: family
tags: [web, family, xss, dom, browser, exfiltration]
skills: [ctf-web]
raw:
  - ../raw/web/xss-dom-and-browser-tricks.md
  - ../raw/misc/RCTF2025-514-wp.md
updated: 2026-07-06
---

# XSS, DOM and Browser Tricks

## 作用边界

本页是浏览器执行与外带 family。共同模式是攻击者需要让浏览器、admin bot、DOM sink、缓存、CSP、MIME、Shadow DOM 或前端框架在受害者上下文中执行或泄露数据。它不再作为单一 technique，因为 XSS payload、DOM clobbering、cache poisoning、XS-Leak 和 admin bot 绕过的证据结构不同。

## 识别信号

- 输入被反射到 HTML、属性、脚本、URL、CSS、Markdown、SVG、模板或前端状态中。
- 题目存在 admin bot、Puppeteer/Playwright 截图、magic link、report endpoint、缓存层、CSP、postMessage 或跨窗口交互。
- 直接读 secret 不可行，但浏览器行为、渲染结果、时间、请求外带或截图回显可作为 oracle。
- sanitizer、MIME sniffing、URL parser、DOM 构造或前端框架状态与服务端过滤不一致。

## 最小证据

- 明确 payload 落点上下文：HTML text、attribute、script string、URL、CSS、DOM property、iframe 或文件 origin。
- 保存 bot 触发条件：是否带 cookie、访问哪个 URL、是否点击、CSP/sandbox/Origin 是什么。
- 对外带类题目，先验证一个 bit、一个字符或一次可控请求能稳定到达攻击者控制端。
- 对截图/file renderer 题目，证明可控内容确实覆盖目标区域或能读取目标 origin 的资源。

## 变体路由

| 信号 | 先确认 | 下一跳 |
|---|---|---|
| 反射/存储 XSS、过滤 `<script>`、Unicode/hex 绕过 | payload 是否进入 HTML/属性/JS/URL/CSS 哪个上下文 | [csp-xsleak-and-browser-exfiltration.md](csp-xsleak-and-browser-exfiltration.md) |
| DOMPurify、DOM clobbering、Shadow DOM、jQuery hashchange | sink、source、sanitize 后结构和浏览器解析结果是否一致 | [csp-xsleak-and-browser-exfiltration.md](csp-xsleak-and-browser-exfiltration.md) |
| admin bot、`javascript:` URL、magic link、redirect chain | bot 是否带 cookie，是否点击/访问攻击者可控 URL，CSP 和跳转链如何限制 | [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md) |
| cache poisoning、Content-Type/MIME mismatch、JPEG+HTML polyglot | 缓存 key、响应头、扩展名和浏览器 MIME sniffing 是否错位 | [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md) |
| CSS font/container query、image timing、postMessage、JSONP/XSSI | 是否能构造稳定 oracle，逐步外带 secret 或状态位 | [csp-xsleak-and-browser-exfiltration.md](csp-xsleak-and-browser-exfiltration.md) |
| JSFuck、AngularJS sandbox、React internal state | 前端框架/编码技巧是否只是到达 sink 的手段，还是题目主障碍 | [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) 或本页 raw |
| Puppeteer/截图渲染器 `file://` | 用户文本是否进入 HTML/JS 模板，页面 origin 是否能读取本地文件，截图是否能作为回显通道 | 先构造最小 iframe/overlay，再回到 Web 首轮记录 bot 触发条件 |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [RCTF2025-514-wp](../raw/misc/RCTF2025-514-wp.md) | Koishi 插件把 text 写进 Puppeteer 渲染页，`file://` origin 允许 iframe 指向本地 flag；payload 重点是绕过 `/` 转义并把 iframe 覆盖到 canvas 上。 |
| [D3CTF2019-babyxss-wp](../raw/web/D3CTF2019-babyxss-wp.md) | XSS/CSP/browser bot 是主线，先确认 sink、CSP 约束和可用 exfil 通道。 |
| [D3CTF2019-d3guestbook-wp](../raw/web/D3CTF2019-d3guestbook-wp.md) | HTML 白名单、JSONP callback 和 CSRF token/sessionid 绑定组合，先构造表单劫持拿管理员 token。 |
| [HGAME2026-博丽神社的绘马挂-wp](../raw/web/HGAME2026-博丽神社的绘马挂-wp.md) | 留言触发 bot 存储型 XSS，归档搜索支持 JSONP callback；让管理员同源上下文查询私有归档并回传。 |
| [LilacCTF2026-playground-wp](../raw/web/LilacCTF2026-playground-wp.md) | 浏览器执行模型或 bot 行为可控，先确认 sink、CSP、触发上下文和 exfil 通道。 |
| [NCTF2026-n-minsite-wp](../raw/web/NCTF2026-n-minsite-wp.md) | MaxSite 源码可由 `/?key` 获取，后台上传入口只校验登录态；上传 HTML/JS 后诱导 bot 读取后台 flag。 |
| [RCTF2025-author-plus-wp](../raw/web/RCTF2025-author-plus-wp.md) | author meta 属性注入过滤了常规空白但漏 `%0b/%0c`，用 CSP meta 压制防护脚本，并借 popover `onbeforetoggle` 在 bot 点击时触发外带。 |
| [RCTF2025-author-wp](../raw/web/RCTF2025-author-wp.md) | 用户名进入 `<meta name=author>` 属性，可注入 CSP 限制 `xss-shield.js`，再在文章正文用 `img onerror` 外带 cookie。 |
| [SU_Note_revWP](../raw/web/SU_Note_revWP.md) | `search.php?q=` 进入内联 JS 字符串且可用 `</script>` 闭合；让 bot 访问内网同源搜索页后 fetch 管理员 notes 并外带。 |

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
- [RCTF2025-514-wp.md](../raw/misc/RCTF2025-514-wp.md)
