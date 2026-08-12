# DownUnderCTF 2021 - JWT

## 题目简述

服务用 RS256 签发包含 `admin: false` 和当前时间的 JWT，却没有公开 RSA 公钥；获取管理员权限需要伪造 `admin: true` 的有效签名。两个已知消息与签名足以恢复模数 $n$，而本题生成的 773 位模数含有极小因子 $29$，可以立即分解并重建私钥。

## 解题过程

### 从签名恢复 RSA 模数

`/get_token` 每次把不同时间写入 payload，因此连续请求可取得两个不同的 RS256 签名。对 PKCS#1 v1.5 签名，若把编码后的消息记为 $m_i$、签名整数记为 $s_i$，公钥指数为常见的 $e=65537$，则：

$$
s_i^e\equiv m_i\pmod n.
$$

所以 $n$ 同时整除：

$$
g_i=s_i^e-m_i.
$$

计算两个 $g_i$ 的最大公约数即可得到 $n$，或得到 $n$ 的一个很小倍数。消息编码必须与 RS256 的 EMSA-PKCS1-v1_5 SHA-256 格式完全一致：

```python
from base64 import urlsafe_b64decode
from hashlib import sha256
from math import gcd

SHA256_INFO = bytes.fromhex(
    "3031300d060960864801650304020105000420"
)

def b64url_decode(value):
    return urlsafe_b64decode(value + "=" * (-len(value) % 4))

def emsa_pkcs1_v1_5(message, size):
    digest_info = SHA256_INFO + sha256(message).digest()
    padding = b"\xff" * (size - len(digest_info) - 3)
    return b"\x00\x01" + padding + b"\x00" + digest_info

def multiple_of_modulus(token, e=65537):
    header, payload, signature = token.split(".")
    sig = b64url_decode(signature)
    s = int.from_bytes(sig, "big")
    encoded = emsa_pkcs1_v1_5(
        f"{header}.{payload}".encode(), len(sig)
    )
    m = int.from_bytes(encoded, "big")
    return pow(s, e) - m

g0 = multiple_of_modulus(token0)
g1 = multiple_of_modulus(token1)
n = gcd(g0, g1)
```

本题两份 token 得到的最大公约数就是实际模数。通用脚本仍应验证候选 $n$ 能否正确校验已有签名，并在必要时移除额外小因子。

### 分解弱模数并重建私钥

恢复的 $n$ 只有 773 位，且直接满足：

```python
assert n % 29 == 0
p = 29
q = n // p
```

于是计算：

$$
\varphi(n)=(p-1)(q-1),\qquad
d\equiv e^{-1}\pmod{\varphi(n)}.
$$

用 PyCryptodome 导出可供 JWT 库使用的 PEM 私钥：

```python
from Crypto.PublicKey import RSA

e = 65537
d = pow(e, -1, (p - 1) * (q - 1))
private_key = RSA.construct((n, e, d, p, q)).export_key()
```

### 伪造管理员 JWT

服务只检查 RS256 签名和 payload 中的 `admin` 布尔值，因此用重建的私钥签发：

```python
import jwt
import requests

forged = jwt.encode(
    {"admin": True},
    private_key,
    algorithm="RS256",
)

result = requests.post(
    BASE_URL + "/get_flag",
    data={"jwt": forged},
)
print(result.text)
```

返回：

```text
DUCTF{json_web_trickeryyy}
```

## 方法总结

本题虽通过 Web 接口交互，决定性障碍是 RSA 签名密钥恢复，因此归入 Crypto。已知 RSASSA-PKCS1-v1_5 消息与签名时，可由 $\gcd(s_1^e-m_1,s_2^e-m_2)$ 恢复模数；真正使伪造成立的是模数含有因子 29，导致私钥可重建。RS256 本身没有被攻破，问题在于密钥生成质量和不安全的 RSA 参数。
