# MiniLCTF2022 CoPiano Writeup

## 题目简述

题目把明文按 32 字节分块。每块随机生成整数 $x$，计算

$$
c=(m\oplus x)^3\bmod N,
$$

并额外公开 $x$ 和 $m\mathbin{\&}x$。原设计可按小根问题处理，但附件实际把 $x$ 也限制在约 256 位，而 $N$ 为 2048 位，导致立方结果根本没有绕模，形成更直接的低指数攻击。

## 解题过程

对每个 256 字节密文块转为整数 $c_i$。由于 $m_i$ 与 $x_i$ 都至多约 256 位，$m_i\oplus x_i$ 也至多 256 位，故

$$
(m_i\oplus x_i)^3<2^{768}<N.
$$

于是模运算没有改变结果，$c_i$ 是一个精确立方数。取整数立方根即可得到 $m_i\oplus x_i$，再异或公开的 $x_i$ 还原明文：

```python
from Crypto.Util.number import long_to_bytes
from gmpy2 import iroot

plain = bytearray()
for c_block, x in zip(cipher_blocks, x_list):
    mixed, exact = iroot(c_block, 3)
    assert exact
    m = int(mixed) ^ x
    plain.extend(long_to_bytes(m, 32))

print(bytes(plain).rstrip(b"\x00"))
```

`t_list=m&x` 在这条非预期路线中不需要使用；它原本用于结合恒等式

$$
m\oplus x=m+x-2(m\mathbin{\&}x)
$$

建立关于小明文 $m$ 的多项式。按附件数据逐块恢复并去掉末尾零填充，得到：

```text
Take a piano... miniLCTF{th3y$4re_n07_1nfin1te.U_@re_!nfinit&!}
```

flag 为：

```text
miniLCTF{th3y$4re_n07_1nfin1te.U_@re_!nfinit&!}
```

## 方法总结

看到裸 RSA 小指数时，第一步应先比较底数上界与模数，而不是直接套 Coppersmith。只要 $M^e<N$，密文就是普通整数幂，精确开根即可。本题也说明冗余泄漏不一定都要使用：先验证最简单的数值边界，往往能发现比预期解更短且更可靠的路径。
