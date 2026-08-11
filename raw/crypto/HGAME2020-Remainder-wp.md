# Remainder

## 题目简述

题目把 RSA 模数拆成三个 1024 位素数 $p,q,r$，分别给出同一密文整数模三个素数的余数。明文长度处于 2048 至 3072 位之间，因此必须先用中国剩余定理把密文恢复到模 $N=pqr$ 的唯一代表，再按多素数 RSA 解密。解出的文本按多行排列，每行只有前两个字符属于 flag。

## 解题过程

已知：

$$
\begin{cases}
c\equiv c_1\pmod p,\\
c\equiv c_2\pmod q,\\
c\equiv c_3\pmod r.
\end{cases}
$$

因为 $p,q,r$ 两两互素，中国剩余定理可在模 $N=pqr$ 下唯一恢复 $c$。多素数 RSA 的欧拉函数为：

$$
\varphi(N)=(p-1)(q-1)(r-1),\qquad d\equiv e^{-1}\pmod{\varphi(N)}.
$$

完整核心代码如下，题目给出的长整数填入对应变量即可：

```python
from math import prod
from Crypto.Util.number import long_to_bytes

def crt(residues, moduli):
    modulus = prod(moduli)
    value = 0
    for residue, current_modulus in zip(residues, moduli):
        partial = modulus // current_modulus
        value += residue * partial * pow(partial, -1, current_modulus)
    return value % modulus

N = p * q * r
c = crt([c1, c2, c3], [p, q, r])
phi = (p - 1) * (q - 1) * (r - 1)
d = pow(e, -1, phi)
message = long_to_bytes(pow(c, d, N)).decode()

# 每行前两个字符是真实片段，其余内容为干扰。
flag = "".join(line[:2] for line in message.splitlines())
print(flag)
```

输出：

```text
hgame{CrT_w0Nt+6Oth3R_mE!!!}
```

## 方法总结

- 核心技巧：先 CRT 合并三个密文余数，再使用 $N=pqr$ 对应的私钥指数解密。
- 识别信号：同一整数的多个互素模余数，且乘积范围足以覆盖目标整数时，应优先使用中国剩余定理。
- 复用要点：CRT 恢复的是模 $N$ 的代表；只有目标已知落在该范围内才可视为唯一整数。解密后的多行干扰还要按题目实际布局逐行取前两个字符。

> 公开 Crypto 复盘补足了原 PDF 未保留的最终明文；本文保留了可独立复算的 CRT 与 RSA 步骤。参考：[HGame 2020 Crypto 题解](https://blog.soreatu.com/posts/writeup-for-crypto-problems-in-hgame-2020/)。
