# CakeCTF 2022 Rock Door Writeup

## 题目简述

服务实现了一种 DSA 风格签名。设 $z=H(m)$，私钥为 $x$，确定性 nonce 为 $k=H(x+z)$，签名满足：

$$
r=H(g^k\bmod p),\qquad
s=(z+xr)k^{-1}\bmod q.
$$

服务允许为任意不含 `goma` 的消息签名，但只返回 $s$，刻意隐藏了 $r$。随后要求提交消息 `hirake goma` 的有效 $(r,s)$。

漏洞来自参数尺度严重不匹配：$q$ 约为 1024 位，而 $x,z,k,r$ 都只是 SHA-256 输出。签名同余式中因此存在一对远小于 $q$ 的整数，可以通过二维格恢复。

## 解题过程

对一条自选消息 $m$ 请求签名，记 $z=H(m)$。由签名公式有：

$$
sk\equiv z+xr\pmod q.
$$

令：

$$
t=z+xr.
$$

因为 $x,r,z$ 都至多 256 位，所以 $t$ 至多约 512 位；$k$ 只有 256 位。与 1024 位的 $q$ 相比，$(k,t)$ 是同余关系：

$$
sk\equiv t\pmod q
$$

中的短整数解。

构造二维格基：

$$
B=\begin{pmatrix}1&s\\0&q\end{pmatrix}.
$$

格中的向量形如 $(u,us+vq)$。取 $u=k$ 并选择合适的 $v$，第二个分量就是 $t$ 或 $-t$。对该格执行 LLL，短向量即可给出 $k$ 与 $t$。官方 solver 根据二者的量级把较小的 256 位数视为 $k$，较大的数视为 $t$。

得到 $k$ 后不需要计算离散对数，可以直接重建：

$$
r=H(g^k\bmod p),\qquad
x=(t-z)r^{-1}\bmod q.
$$

随后按服务端算法为目标消息自行签名：

```python
from hashlib import sha256
from Crypto.Util.number import long_to_bytes

def h(data):
    return int.from_bytes(sha256(data).digest(), "big")

def recover_short_pair(s, q):
    basis = matrix(ZZ, [[1, s], [0, q]])
    short = basis.LLL()[0]
    a, b = abs(int(short[0])), abs(int(short[1]))
    return (a, b) if a < b else (b, a)  # k, t

# m 为先前送去签名的普通消息，s 为服务返回值。
z = h(m)
k, t = recover_short_pair(s, q)
r = h(long_to_bytes(pow(g, k, p)))
x = (t - z) * pow(r, -1, q) % q

target = b"hirake goma"
z2 = h(target)
k2 = h(long_to_bytes(x + z2))
r2 = h(long_to_bytes(pow(g, k2, p)))
s2 = (z2 + x * r2) * pow(k2, -1, q) % q
```

提交 `hirake goma`、$r_2$ 和 $s_2$ 后得到：

```text
CakeCTF{uum_hash_is_too_small_compared_to_modulus}
```

## 方法总结

隐藏签名分量并不能修复参数设计错误。这里真正泄露私钥的是 $sk\equiv z+xr\pmod q$ 中的两个短量 $k$ 与 $z+xr$；二维 LLL 将它们从大模数同余中恢复出来，再由公开算法重建 $r$ 和 $x$。

设计签名方案时，nonce、哈希映射和群阶必须处在匹配的安全尺度，并应采用经过审计的标准方案。自定义地截断哈希、隐藏输出字段或使用确定性 nonce，都不能替代正确的安全证明。
