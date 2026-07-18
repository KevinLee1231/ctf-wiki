# week3Wilson

## 题目简述

题目令 $p$ 为 512 bit 素数、$q=\operatorname{next\_prime}(p)$，并构造

$$
\text{noise}=(p-1)!\bmod q,qquad
m=\text{noise}\cdot\text{flag}\bmod n,qquad n=pq.
$$

随后对 $m$ 做 RSA 加密。目标是利用近素数分解和 Wilson 定理消去 `noise`。

## 解题过程

因为 $p,q$ 相邻，$n$ 的两个因子非常接近，可用 Fermat 分解。恢复私钥并解密后有

$$
m\equiv\text{noise}\cdot\text{flag}\pmod q.
$$

Wilson 定理给出 $(q-1)!\equiv-1\pmod q$，而

$$
(q-1)!=(p-1)!\prod_{i=p}^{q-1}i.
$$

记 $P=\prod_{i=p}^{q-1}i$，则 $\text{noise}\cdot P\equiv-1\pmod q$，从而

$$
\text{flag}\equiv-mP\pmod q.
$$

由于源码注明 flag 只有 37 字节，小于 512 bit 的 $q$，该剩余类就是 flag 整数本身。

```python
import re
from math import isqrt
from pathlib import Path
from Crypto.Util.number import inverse, long_to_bytes

text = Path("task.py").read_text(encoding="utf-8")
n = int(re.search(r"# n=(\d+)", text).group(1))
c = int(re.search(r"# c=(\d+)", text).group(1))

# Fermat 分解：n = A^2 - B^2 = (A-B)(A+B)
A = isqrt(n)
if A * A < n:
    A += 1
while True:
    B2 = A * A - n
    B = isqrt(B2)
    if B * B == B2:
        p, q = A - B, A + B
        break
    A += 1

d = inverse(65537, (p - 1) * (q - 1))
m = pow(c, d, n)

product = 1
for value in range(p, q):
    product = product * value % q
flag = (-m * product) % q
print(long_to_bytes(flag).decode())
```

得到：

```text
0xGame{n0w_y0u_kn0w_Wi1s0n's_the0rem}
```

## 方法总结

相邻素数构成的 RSA 模数适合 Fermat 分解。面对无法直接计算的超大阶乘，应寻找 Wilson 定理等模意义恒等式；本题只需计算很短的区间乘积 $p\cdots(q-1)$，即可消去 $(p-1)!$。
