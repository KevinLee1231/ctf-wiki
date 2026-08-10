# ezRSA

## 题目简述

题目给出两个 309 位整数 `leak1`、`leak2`，以及 617 位 RSA 密文 `c`，公开指数为 `0x10001`。两个 leak 实际就是模数的素因子 $p$、$q$，因此不需要分解 $n$；直接计算欧拉函数和私钥指数即可解密。

## 解题过程

令：

$$
n=pq,
\qquad
\varphi(n)=(p-1)(q-1),
\qquad
d\equiv e^{-1}\pmod{\varphi(n)}.
$$

RSA 解密为：

$$
m\equiv c^d\pmod n.
$$

题目原始的三项大整数只作为输入数据，正文不重复堆放；将其代入以下脚本即可：

```python
from Crypto.Util.number import inverse, long_to_bytes

# 替换为题目给出的两个 309 位 leak 和 617 位密文。
p = leak1
q = leak2
ciphertext = c

e = 0x10001
n = p * q
phi = (p - 1) * (q - 1)
d = inverse(e, phi)

plaintext = long_to_bytes(pow(ciphertext, d, n))
print(plaintext)
```

对 PDF 中的原始数值执行该计算，得到：

```text
hgame{F3rmat_l1tt1e_the0rem_is_th3_bas1s}
```

官方题解把考点概括为“费马小定理”；更准确地说，本题利用的是已知 $p,q$ 后可计算 $\varphi(n)$，再由模逆求出 RSA 私钥指数。这里不存在对大模数的实际因数分解。

## 方法总结

- 核心技巧：识别泄露值就是 RSA 素因子，直接重建私钥。
- 识别信号：题目同时给出两个与 RSA 模数规模匹配的大整数，二者乘积可作为 $n$，而公开指数为常见的 $65537$。
- 复用要点：拿到候选因子后应检查 $n=pq$、$\gcd(e,\varphi(n))=1$，并验证解密结果具有预期 flag 前缀。
