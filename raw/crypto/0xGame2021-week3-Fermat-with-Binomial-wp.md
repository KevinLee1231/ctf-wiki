# week3Fermat with Binomial

## 题目简述

附件给出 RSA 参数 $n,c$ 和两个提示：

$$
\begin{aligned}
h_1&=(2021p+q)^E\bmod n,\\
h_2&=(1010p+1011)^q\bmod n,
\end{aligned}
$$

其中 $E=20212021$、$n=pq$。目标是利用模运算恢复素因子并解密。

## 解题过程

先在模 $q$ 下观察两个提示。由二项式展开，$h_1$ 中所有含 $q$ 的项都消失：

$$
h_1\equiv(2021p)^E\pmod q.
$$

对任意整数 $x$，都有 $x^q\equiv x\pmod q$，因此

$$
h_2\equiv1010p+1011\pmod q.
$$

于是

$$
(h_2-1011)^E\cdot2021^E-h_1\cdot1010^E\equiv0\pmod q.
$$

该差值必含因子 $q$，而模 $p$ 时通常不为零，所以与 $n$ 求最大公因数即可得到 $q$。原 WP 把两个幂直接写成“底数之差的幂”，这在一般情况下不成立；上面的分别模 $q$ 推导才是严格依据。

```python
from math import gcd
from pathlib import Path
from Crypto.Util.number import inverse, long_to_bytes

values = {}
for line in Path("message.txt").read_text().splitlines():
    name, value = line.split("=", 1)
    values[name] = int(value)

n = values["n"]
c = values["c"]
h1 = values["hint1"]
h2 = values["hint2"]
E = 20212021

difference = (
    pow(h2 - 1011, E, n) * pow(2021, E, n)
    - h1 * pow(1010, E, n)
) % n
q = gcd(n, difference)
if q in (1, n):
    raise ValueError("未得到非平凡因子")
p = n // q

private = inverse(65537, (p - 1) * (q - 1))
plain = long_to_bytes(pow(c, private, n))
print(plain.decode())
```

得到：

```text
0xGame{F3rm4t_with_Bin0mi41_th30r3m}
```

## 方法总结

含 $p,q$ 的 RSA hint 应分别在模 $p$、模 $q$ 下化简。二项式展开可消去含某个素因子的项，Fermat 同余可降低指数；构造出“模某素因子为零”的表达式后，用 `gcd` 提取该因子。
