# e

## 题目简述

题目仍是低指数 RSA，但这次 $e=5$，明文由已知前缀 `the challenges flag is tjctf{` 和较短未知后缀组成。明文五次方已经超过模数，不能直接求整数五次根；已知高位与短未知低位构成一元 Coppersmith 小根问题。

题目生成器是 Sage 文件，其中 `^` 表示乘方而不是 Python 的按位异或，因此密文确实是 $m^5\bmod N$。

## 解题过程

设未知部分占 $k$ 位，已知前缀对应整数为 $a$，则完整明文可写成

$$
m=a\cdot2^k+x,\qquad 0\le x<2^k.
$$

在 $\mathbb Z_N$ 上建立首一多项式

$$
f(x)=(a\cdot2^k+x)^5-C,
$$

其小根就是未知后缀。官方脚本枚举 5 到 14 个未知字节，并用 Howgrave-Graham 形式的 Coppersmith/LLL 求根；核心调用可以简化为：

```sage
from Crypto.Util.number import bytes_to_long, long_to_bytes

known = bytes_to_long(b"the challenges flag is tjctf{")
R.<x> = PolynomialRing(Zmod(N))

for unknown_bytes in range(5, 15):
    k = 8 * unknown_bytes
    f = (known * 2^k + x)^5 - C
    roots = f.small_roots(X=2^k, beta=1)
    for root in roots:
        candidate = long_to_bytes(known * 2^k + int(root))
        if candidate.startswith(b"the challenges flag is tjctf{"):
            print(candidate)
```

恢复出的消息后缀给出：

```text
tjctf{coppersword2}
```

## 方法总结

- “低指数”不只对应整数开根；当明文有已知高位、未知低位较短时，应转向 Coppersmith 小根攻击。
- 建模时要明确未知部分位于高位还是低位，并让界 $X$ 与未知字节数一致。
- `.sage` 中 `^` 的语义不同于普通 Python；阅读生成器时必须先确认实际解释器，避免把乘方误判成 XOR。
