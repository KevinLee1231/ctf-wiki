# bi0sCTF 2022 - LEAKY DSA

## 题目简述

服务在 NIST P-256 曲线上实现了一套 ECDSA 风格的签名。私钥 $d$ 直接由 flag 字节转成整数；每次签名返回消息摘要 $z$、签名 $(r,s)$，以及将 nonce $k$ 的低 120 位清零后的值：

$$
K=\left\lfloor\frac{k}{2^{120}}\right\rfloor2^{120}.
$$

攻击者可以提交两条消息。目标是利用两次 nonce 的高位泄漏，恢复共同的私钥 $d$。

## 解题过程

### 写出两次签名的私钥表达式

签名满足：

$$
s=k^{-1}(z+dr)\pmod q,
$$

因此：

$$
d=(sk-z)r^{-1}\pmod q.
$$

分别请求两条不同消息，记录服务返回值：

```python
io.sendlineafter(b"Enter message: ", b"a")
z1, r1, s1, K1 = eval(io.recvline())

io.sendlineafter(b"Enter message: ", b"b")
z2, r2, s2, K2 = eval(io.recvline())
```

令未知低位为 $x,y$：

$$
k_1=K_1+x,\quad k_2=K_2+y,\qquad 0\le x,y<2^{120}.
$$

两次签名使用同一个 $d$，于是可在 $\mathbb Z_q$ 上建立二元多项式：

$$
f(x,y)=
(s_1(K_1+x)-z_1)r_1^{-1}
-(s_2(K_2+y)-z_2)r_2^{-1}
\equiv0\pmod q.
$$

Sage 建模如下：

```python
PR.<x, y> = PolynomialRing(Zmod(q), 2)

f = ((s1 * (K1 + x) - z1) * inverse_mod(r1, q)
     - (s2 * (K2 + y) - z2) * inverse_mod(r2, q))
```

### 用格约化恢复低位

虽然 $f$ 只有一条模方程，但 $x,y$ 都有严格的小界。官方 `small_roots` 实现对 $N^{m-i}f^i$ 乘以若干单项式移位，将系数按 $(2^{120},2^{120})$ 缩放后执行 LLL；短向量对应在整数域上也为零的多项式。逐个加入这些多项式，直到生成的理想是零维，再用 Gröbner 基/`variety` 求共同整数根：

```python
roots = small_roots(
    f,
    bounds=(2^120, 2^120),
    m=1,
    d=4,
)
x0, y0 = roots[0]

assert 0 <= int(x0) < 2^120
assert 0 <= int(y0) < 2^120
```

恢复 $x$ 后代回任一签名：

```python
from Crypto.Util.number import long_to_bytes

d = (s1 * (K1 + x0) - z1) * inverse_mod(r1, q) % q
print(long_to_bytes(int(d)))
```

输出为：

```text
bi0sctf{3CC_S1gn1nG_1s_SECCY_6675636b}
```

可以再用第二组签名验证恢复值：

```python
d2 = (s2 * (K2 + y0) - z2) * inverse_mod(r2, q) % q
assert d == d2
```

## 方法总结

ECDSA 的安全性不仅要求 nonce 不重复，还要求 nonce 的每一位都足够隐蔽。每次泄漏高位后，未知部分只有 120 位；两次签名把同一个私钥表示成两个关于小未知量的线性式，消去私钥便得到适合格攻击的二元小根问题。验证阶段应同时检查低位范围和两次计算出的私钥是否一致，避免把 LLL 产生的伪根误当成结果。
