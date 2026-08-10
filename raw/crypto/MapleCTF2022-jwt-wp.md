# jwt

## 题目简述

服务使用自制的 ES256 实现签发 JWT。审计签名代码可以发现，它把 ECDSA 的随机 nonce $k$ 直接设为私钥 $d$。这不是普通的 JWT 算法混淆或密钥类型混用，而是 ECDSA nonce 与私钥之间出现了可直接求解的代数关系。

## 解题过程

ECDSA 签名满足

$$
r=x([k]G)\bmod n,\qquad s=k^{-1}(z+rd)\bmod n,
$$

其中 $z$ 是 `header.payload` 的 SHA-256 整数。题目令 $k=d$，于是

$$
s=d^{-1}(z+rd)=zd^{-1}+r\pmod n,
$$

从而

$$
d=z(s-r)^{-1}\pmod n.
$$

注册任意普通用户，取得一个合法 token。按题目实现而不是第三方库的默认格式解析签名：源码把 32 字节的 $r$ 与 $s$ 以小端序拼接。随后计算私钥：

```python
signature = b64url_decode(token.split(".")[2])
r = int.from_bytes(signature[:32], "little")
s = int.from_bytes(signature[32:], "little")
z = bytes_to_long(sha256(signing_input).digest())
d = z * inverse(s - r, curve_order) % curve_order
```

用恢复出的 $d$ 签发 `{"user":"admin"}`，替换 cookie 后访问管理员页面，得到：

```text
maple{3ll1pt!c_c2rv3s_f7w!!!}
```

## 方法总结

ECDSA 的 $k$ 必须每次独立、均匀且不可预测；nonce 重用、可预测或与私钥相关都会把签名方程变成私钥泄露。本题只需一个签名，因为关系是 $k=d$，不需要等待 nonce 重复。复现时还必须跟随题目自制序列化的端序，否则即使代数推导正确也无法得到有效私钥。
