# Cycles

## 题目简述

题目给出素数 $p$、底数 $g=3$、提示 $g^a\bmod p=1$ 和 AES-ECB 密文。隐藏参数的生成方式是 $a=(p-1)N$，其中 $N$ 是 5 到 50 的小整数；AES 密钥取 `long_to_bytes(a)[:16]`。

提示值为 1 并非离散对数泄露，而是在提示指数位于乘法群阶的倍数。

## 解题过程

由费马小定理，对任意不被 $p$ 整除的 $g$ 都有

$$
g^{p-1}\equiv1\pmod p.
$$

因此令 $a=i(p-1)$ 并枚举小整数 $i$ 即可。每次按服务端完全相同的方式取整数的大端字节前 16 字节作为 AES 密钥；解密后检查 `byuctf{` 前缀并去除 PKCS#7 填充。

```python
from Crypto.Cipher import AES
from Crypto.Util.number import long_to_bytes
from Crypto.Util.Padding import unpad

for i in range(1, 100):
    a = i * (p - 1)
    key = long_to_bytes(a)[:16]
    pt = AES.new(key, AES.MODE_ECB).decrypt(ciphertext)
    if b"byuctf{" in pt:
        print(unpad(pt, 16))
        break
```

在很小的搜索范围内即可恢复：

```text
byuctf{1t_4lw4ys_c0m3s_b4ck_t0_1_21bcd6}
```

## 方法总结

- 核心技巧：把 $g^a\equiv1\pmod p$ 与群阶 $p-1$ 联系起来，枚举小倍数而非求离散对数。
- 识别信号：素数模数、提示值 1、指数长度受限或由小参数生成时，应优先检查指数是否为群阶倍数。
- 复用要点：密钥派生必须复刻实现中的字节序和截断位置；错误地取末 16 字节会得到完全不同的 AES 密钥。
