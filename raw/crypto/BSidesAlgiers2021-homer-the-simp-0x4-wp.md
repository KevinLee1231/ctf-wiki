# Homer The Simp - 0x4

## 题目简述

前一阶段获得的 `love_letter.txt` 包含三行 RSA 数据，每行都是：

```text
十进制模数 n:Base64(ciphertext)
```

三组公钥都使用 $e=65537$。模数表面上足够大，但生成时只使用了三个素数，并让三个模数两两共享其中一个素因子：

```text
n1 = p1 * p2
n2 = p1 * p3
n3 = p2 * p3
```

RSA 模数之间共享素因子会使分解退化为一次最大公约数运算。

## 解题过程

对三个模数做两两 GCD：

```text
gcd(n1, n2) = p1
gcd(n1, n3) = p2
gcd(n2, n3) = p3
```

这样便能分别构造三组欧拉函数和私钥指数：

$$
\varphi(n_i)=(p-1)(q-1),\qquad d_i=e^{-1}\bmod\varphi(n_i)
$$

密文先从 Base64 解码为大端整数，再计算 $m_i=c_i^{d_i}\bmod n_i$。三段明文按文件中的顺序拼接即可。完整求解脚本如下：

```python
#!/usr/bin/env python3
from base64 import b64decode
from math import gcd


e = 65537
rows = []

for line in open("love_letter.txt", encoding="utf-8"):
    modulus, ciphertext = line.strip().split(":", 1)
    rows.append(
        (
            int(modulus),
            int.from_bytes(b64decode(ciphertext), "big"),
        )
    )

n1, n2, n3 = (row[0] for row in rows)
p1 = gcd(n1, n2)
p2 = gcd(n1, n3)
p3 = gcd(n2, n3)

factor_pairs = (
    (p1, p2),
    (p1, p3),
    (p2, p3),
)

plaintext = bytearray()
for (n, ciphertext), (p, q) in zip(rows, factor_pairs):
    assert p * q == n
    phi = (p - 1) * (q - 1)
    d = pow(e, -1, phi)
    message = pow(ciphertext, d, n)
    plaintext.extend(message.to_bytes((message.bit_length() + 7) // 8, "big"))

print(bytes(plaintext).rstrip(b"\x00").decode())
```

输出是一封分成三段的明文信件，末尾给出：

```text
shellmates{m1sS10n_4cC0mplisHed_t1m3_t0_g0_h0m3}
```

## 方法总结

本题是 RSA 批量密钥生成中的共享素数问题。只要两个模数复用了一个素数，$gcd(n_i,n_j)$ 就会直接泄露该因子；大模数和常用公开指数都无法弥补密钥生成失败。

看到多组 RSA 模数时，应把 pairwise GCD 放在低成本首检中。真实批量场景还可以用乘积树与余数树加速，而单题只有少量模数时，两两 GCD 已足够。恢复明文后还要保持原始密文顺序，并注意整数转字节时可能出现应用层附加的尾随空字节。
