# SQL Injection

## 题目简述

登录接口把用户提交的 `username` 直接拼进 PostgreSQL 查询，异常消息又通过 Flask `flash` 原样显示。数据库初始化脚本把第一阶段 flag 直接存为 `admin` 的 `password_hash`，因此可以利用错误回显让 PostgreSQL 把该字段内容带入类型转换异常。

## 解题过程

危险查询是：

```python
cur.execute(
    f"SELECT id, password_hash FROM users WHERE username='{username}'"
)
```

在用户名中提交：

```sql
admin' AND CAST(password_hash AS INTEGER)=1--
```

拼接后，数据库会对 `admin` 行的 `password_hash` 做整数转换。该值实际是 flag，不是数字，于是 PostgreSQL 抛出 `invalid input syntax for type integer`，异常文本包含原始字段值。应用捕获异常后把它显示在登录页，从回显中读出：

```text
grey{this_is_a_database_secret}
```

密码字段可填任意非空内容，因为查询在执行 `check_password_hash` 前已经抛出异常。

## 方法总结

SQL 注入不只用于认证绕过。若应用把数据库异常直接回显，可以主动把文本秘密送进不兼容的类型转换，利用错误消息完成数据外带。修复应同时采用参数化查询、正确的密码哈希存储，并停止向客户端暴露原始数据库异常。
