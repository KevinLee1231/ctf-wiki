# Hurtful

## 题目简述

题目采用 2048 位 RSA 模数和很小的公钥指数 $e=3$，没有任何随机填充。明文由一段完全已知的长前缀与未知 flag 直接拼接：

```python
message = b"Congrats on making it all the way here. ... just find the flag: " + flag
c = pow(bytes_to_long(message), 3, N)
```

这构成 RSA stereotyped message 场景：明文大部分已知，未知后缀足够短，可用 Coppersmith 小根算法恢复。

## 解题过程

若假设未知后缀长度为 $\ell$ 字节，令

$$
a=\operatorname{bytes\_to\_long}(\text{prefix})\cdot 2^{8\ell},
$$

未知 flag 对应整数 $x$，则完整明文为 $m=a+x$，并满足

$$
f(x)=(a+x)^3-c\equiv0\pmod N.
$$

因为 $x$ 相对 $N$ 足够小，可以在 $\mathbb Z_N[x]$ 上求一元小根。题面没有直接给出 flag 长度，所以从合理下界开始枚举 $\ell$；长度猜对时才能把已知前缀移到正确的高位。

```sage
from Crypto.Util.number import bytes_to_long, long_to_bytes

def recover(prefix, secret_len, c, n):
    Z = Zmod(n)
    R.<x> = PolynomialRing(Z)
    a = Z(bytes_to_long(prefix) * 2^(8 * secret_len))
    f = ((a + x)^3 - c).monic()
    roots = f.small_roots(epsilon=1/20)
    if len(roots) == 1:
        return long_to_bytes(int(a + roots[0]))

for secret_len in range(10, 41):
    result = recover(prefix, secret_len, c, N)
    if result is not None:
        print(result)
        break
```

恢复出的完整明文末尾为：

```text
byuctf{cuz_st3r30typ3s_hurt_92de04}
```

## 方法总结

- 核心技巧：把“已知前缀 + 短未知后缀”的低指数裸 RSA 写成模多项式，并用 Coppersmith 求小根。
- 识别信号：$e=3$、无 OAEP/PKCS#1 填充、固定长前缀和短秘密字段同时出现时，应优先考虑 stereotyped message attack。
- 复用要点：未知字段长度决定前缀左移位数；长度未知时可以有界枚举。恢复结果还应检查已知前缀和 flag 格式，不能把任意小根当作答案。
