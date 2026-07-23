# EnD

## 题目简述

题目由 Node.js URL 代理、管理员 bot 和只监听在管理员本机视角下的 Flask 消息 API 组成。`/admin` 需要 bot 的 `session` Cookie，页面中直接显示内部 API 地址和 16 位 API key；API 的 inbox 只有一条消息，即环境变量 `_SECRET` 中的 flag。

常规 SSRF 路径被多层限制：新增页面只接受带点号的公网主机，解析后的 IP 不能是私网；代理转发时还会删除 Cookie、Referer，并对所有响应附加 `script-src 'self'` 和 `nosniff`。预览字段经过 DOMPurify，因此普通 XSS 也走不通。

## 解题过程

### 利用 `Content-Length: 0` 制造响应错位

先把攻击者服务器注册成一个代理页面，再让 bot 访问攻击者的 `/trigger`。该页面弹出代理下的 `/view/<name>/`，后者从攻击者站点加载 8 个并行的异步脚本，为同一前端连接制造后续请求。

漏洞位于代理处理脚本响应的逻辑：

```javascript
if (req.headers["sec-fetch-dest"] === "script") {
    h["content-length"] = "0"
    delete h["transfer-encoding"]
}
res.writeHead(proxyRes.statusCode, proxyRes.statusMessage, h)
proxyRes.pipe(res)
```

它把发往浏览器的 `Content-Length` 改为 0，却仍继续转发上游 body。攻击者令 `s6.js` 的上游 body 恰好是一份完整的第二响应：

```http
HTTP/1.1 200 OK
Content-Type: application/javascript
Content-Length: <payload length>
Connection: keep-alive

<JavaScript payload>
```

浏览器把第一个脚本响应视为空响应，连接中多出的字节则与池中的后续脚本请求对齐，最终把伪造响应作为同源 JavaScript 执行。这是前后端对响应边界理解不一致造成的客户端响应错位；CSP 允许它，是因为脚本响应在浏览器看来来自代理自身。

### 读取管理员配置

错位脚本运行在代理 origin，可自动携带 bot 预先设置的 HttpOnly `session` Cookie：

```javascript
const page = await fetch('/admin', {
  credentials: 'include'
}).then(r => r.text())
```

从 HTML 中提取 `#api-url` 和 `#api-key` 后，脚本打开攻击者的 oracle 页面，并把这两个值作为查询参数传过去。此时已经拿到鉴权材料，但 oracle 页面仍不能直接读取 `http://localhost:9090/messages/search`：API 没有开放 CORS。

### 用媒体 Range 和 Service Worker 构造前缀 oracle

搜索接口执行 `m.startswith(q)`，并通过 Flask `send_file(..., conditional=True)` 返回 JSON，因此支持 Range。若候选前缀命中，JSON 中包含整条长 secret；若不命中，响应只有空结果数组。令阈值 $T=30$，两者对 `Range: bytes=30-` 分别产生可满足的部分响应和越界响应。

oracle 页面先注册 Service Worker，再创建 `<audio>` 指向内部搜索 URL。Service Worker 对媒体首次请求 `Range: bytes=0-` 返回一个伪造的 30 字节 `206` 和超大总长度，诱使媒体栈继续请求 `Range: bytes=30-`；第二次请求才被转发到真实内部 API。成功时保存跨源 opaque `Response`，随后在 `<audio>` 的错误回调里把它移作同源 `/mock.css` 的响应。浏览器是否拒绝这个响应替换会反映第二段 Range 是否成功，从而得到一位布尔值，而脚本始终没有直接读取跨源正文。

因此可实现：

```text
oracle(prefix + candidate) -> true / false
```

依次枚举 `_0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ{}!@#$%&-`，每确认一字符便回传攻击者服务器。bot 单次生命周期可能不够恢复全部内容，`solver.js` 会保存最长前缀，并在最多三轮 bot 提交之间继续。

最终泄露：

```text
SEKAI{proxy_said_n0_w4y_l0ng_W4y_4nD_f1n4lllyyy_Y0u_are_H3r3_here_eda1ndj}
```

## 方法总结

本题串联了三个不同信任边界：代理的响应定界错误把攻击者代码送入管理员 origin，同源权限泄露内部 API 配置，媒体 Range 与 Service Worker 又把不可读的跨源响应压缩成一位前缀 oracle。这里并非传统“缓存命中”攻击，关键是保存并移用 opaque `Response` 后产生的成功/失败差异。CSP、私网过滤和 CORS 各自只覆盖一层，不能替代端到端的消息边界与浏览器行为审计。
