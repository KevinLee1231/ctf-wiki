# week3签个名吧

## 题目简述

题目给出两条 DSA 签名，消息分别为 `0xGame` 和 `hack_fun`。两条签名的 $r$ 完全相同，源码也确认签名对象只在初始化时生成一次随机数 $k$，随后重复使用。DSA 一旦复用 nonce，就可以由两条签名恢复 $k$ 和长期私钥 $x$；flag 是私钥十进制字符串的 MD5。

## 解题过程

DSA 签名满足：

$$
s_i\equiv k^{-1}(z_i+xr)\pmod q,
$$

其中 $z_i$ 是消息的 SHA-256 摘要按大端序转成的整数。两式相减可消去私钥项：

$$
k\equiv(z_0-z_1)(s_0-s_1)^{-1}\pmod q.
$$

再代回任意一条签名：

$$
x\equiv(s_0k-z_0)r^{-1}\pmod q.
$$

```python
from hashlib import md5, sha256

from Crypto.Util.number import bytes_to_long, inverse

q = 1427665647738374763020227949129429759446792665193
r = 9569108440001628337054549116871993930089020799
s0 = 1155391566683353144613828381835889947132557976718
s1 = 182166581822791423481695372664923137176789829383

m0 = b"0xGame"
m1 = b"hack_fun"
z0 = bytes_to_long(sha256(m0).digest())
z1 = bytes_to_long(sha256(m1).digest())

k = ((z0 - z1) * inverse((s0 - s1) % q, q)) % q
x = ((s0 * k - z0) * inverse(r, q)) % q

flag = "0xGame{" + md5(str(x).encode()).hexdigest() + "}"
print("x =", x)
print(flag)
```

输出：

```text
x = 664253418806374654523250549731294350835072393473
0xGame{0d49c454e403b622070f7257682fe8d6}
```

## 方法总结

DSA 的随机数 $k$ 必须对每次签名唯一且不可预测。相同 $k$ 会产生相同 $r$，并让两条线性同余方程足以恢复 nonce 和私钥。复现时要保持所有减法与求逆都在模 $q$ 下进行，并按题目源码使用 SHA-256 摘要的整数值；最后 MD5 的输入是 `str(x)`，不是私钥的定长二进制编码。
