# ezMath

## 题目简述

题目给出 $N=114514$ 和一段 64 字节 AES-ECB 密文。AES 密钥不是直接提供的，而是由 Pell 方程

$$
x^2-Ny^2=1
$$

的最小正整数解生成：把 $y$ 转为大端字节串，按零字节补齐到 16 字节倍数，再取前 16 字节作为密钥。

## 解题过程

对 $\sqrt N$ 做连分数展开，其收敛分数会依次给出 $x/y$ 的候选值。找到第一个满足 $x^2-Ny^2=1$ 的候选，即得到基本解。完整求解脚本如下：

```python
from math import isqrt

from Crypto.Cipher import AES
from Crypto.Util.number import long_to_bytes

N = 114514
ciphertext = bytes.fromhex(
    "cef19484e96d8804cb9a649e0862bf8b"
    "d30de28117679cd710191aa6c39ddee7"
    "e068ed2f0095747a29315c09383ab12c"
    "55fede63f268ab60e52793f8deb29a9a"
)


def solve_pell(n: int) -> tuple[int, int]:
    a0 = isqrt(n)
    if a0 * a0 == n:
        raise ValueError("N 不能是完全平方数")

    m = 0
    denominator = 1
    a = a0

    p_prev2, p_prev1 = 0, 1
    q_prev2, q_prev1 = 1, 0

    while True:
        x = a * p_prev1 + p_prev2
        y = a * q_prev1 + q_prev2
        if x * x - n * y * y == 1:
            return x, y

        p_prev2, p_prev1 = p_prev1, x
        q_prev2, q_prev1 = q_prev1, y
        m = denominator * a - m
        denominator = (n - m * m) // denominator
        a = (a0 + m) // denominator


_, y = solve_pell(N)
key_material = long_to_bytes(y)
key_material += b"\x00" * ((16 - len(key_material) % 16) % 16)
key = key_material[:16]

plaintext = AES.new(key, AES.MODE_ECB).decrypt(ciphertext)
print(plaintext.rstrip(b"\x00").decode())
```

基本解中的 $y$ 为：

```text
9037815138660369922198555785216162916412331641365948545459353586895717702576049626533527779108680
```

由其得到的 AES 密钥十六进制为 `043b30bec7ca05f923d74170c4c9be19`，解密结果是：

```text
hgame{G0od!_Yo3_k1ow_C0ntinued_Fra3ti0ns!!!!!!!}
```

## 方法总结

- 核心技巧：用 $\sqrt N$ 的连分数收敛分数求 Pell 方程基本解，再按题目定义派生 AES 密钥。
- 识别信号：出现 $x^2-Ny^2=1$、非平方整数 $N$，以及要求从解的某一分量构造密钥。
- 复用要点：不能只求任意解，通常需要最小正解；密钥派生中的字节序、补零位置和截断顺序必须逐字复现，否则 AES 解密不会得到结构化明文。
