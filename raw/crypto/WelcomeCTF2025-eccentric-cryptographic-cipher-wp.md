# Eccentric Cryptographic Cipher

## 题目简述

题目使用同一个 RSA 模数 $(n,e)$ 加密多条消息，并公开除 flag 外各条消息的明文。关键消息不是普通已知明文，而是 $m=q+1$，其中 $q$ 是模数的一个素因子。它在模 $q$ 意义下等于 1，因此 RSA 加密后仍满足 $c\equiv1\pmod q$，形成可用于分解模数的固定点。

## 解题过程

对每条密文 $c$ 计算：

$$
g=\gcd(c-1,n)
$$

当对应明文是 $q+1$ 时：

$$
c=(q+1)^e\equiv1^e\equiv1\pmod q
$$

所以 $q\mid(c-1)$，而该关系通常不同时模 $p$ 成立，于是 $1<g<n$ 且 $g=q$。官方 solver 同时检查 `c + 1`，也可覆盖模某因子为 $-1$ 的固定点。

```python
import json
from math import gcd
from Crypto.Util.number import long_to_bytes

data = json.load(open("chall.json", encoding="utf-8"))
n, e, entries = data["n"], data["e"], data["encs"]

factor = None
for entry in entries:
    ciphertext = entry["ct"]
    for candidate in (ciphertext - 1, ciphertext + 1):
        value = gcd(candidate, n)
        if 1 < value < n:
            factor = value
            break
    if factor is not None:
        break

p, q = factor, n // factor
phi = (p - 1) * (q - 1)
d = pow(e, -1, phi)

for entry in entries:
    plaintext = long_to_bytes(pow(entry["ct"], d, n))
    if plaintext.startswith(b"grey{"):
        print(plaintext.decode())
```

得到：

```text
grey{d3j4_vu_b3caus3_1v3_pl4y3ed_th3s3_g4m3s_b4}
```

## 方法总结

- 核心技巧：利用 RSA 消息在某个 CRT 分量上的 $\pm1$ 固定点，通过 $\gcd(c\mp1,n)$ 泄露非平凡因子。
- 识别信号：同模数加密多条消息，题面强调“某条消息根本没有被加密”，并给出大量已知明文作为掩护。
- 复用要点：固定点不只包括全局的 0、1、$-1$；只要消息在模某一素因子下固定，就可能用与密文的差做 GCD 分解模数。
