# homo

## 题目简述

附件 `gentry.py` 实现了一个近似 GCD 型的简化 Gentry 同态加密玩具方案。源码中私钥 `sk` 是 514 bit 素数 $q$，公钥由 981 个样本 $p_iq+2r_i$ 组成，其中 $p_i$ 约 114 bit、扰动 $r_i$ 约 191 bit；密文逐 bit 加密，每个 bit 都是明文位加上若干公钥样本的随机子集和。因为参数随意选取，公钥样本足以通过 AGCD 格攻击恢复等价私钥 $q$，再逐 bit 解密 flag。

## 解题过程

题目实现的是gentry的同态加密方案，但是参数是随意选取的，这就导致该方案能够使用公钥直接计算出等价私钥。

首先，该方案的流程如下

密钥生成：
$$
sk=q,\quad pk=\{p_iq+2r_i\}_{i=0}^n
$$
其中，p，q，r均为素数。

加密时，要求明文仅有1bit
$$
d_i\leftarrow\{0,1\},\quad c=m+\sum_{i=0}^n d_i pk_i
$$
解密时，计算
$$
m=(c\bmod sk)\bmod 2
$$
由于 $pk_i\bmod sk=2r_i$ 为偶数，因此模 $sk$ 后，$m$ 即为结果的 LSB。

由于多个 $pk_i$ 均为 $q$ 的倍数加上小扰动（$r_i$ 相对于 $q$ 较小），因此可以使用求解 AGCD 的方法恢复私钥 $q$。

AGCD 参考资料中的关键思想是：样本形如 $x_i=q_i p+r_i$，若噪声 $r_i$ 足够小，则存在由样本线性组合得到的短向量；构造格并做 LLL 后，短向量可泄露公因子或等价私钥。本题就是把这个思路用到 $pk_i=p_iq+2r_i$ 上。

```python
from sage.all import *
from Crypto.Util.number import *
c = 
pubkey = 
rbit = 191
Mlen = 10
M = Matrix(ZZ , Mlen , Mlen)
for i in range(1,Mlen):
    M[0 , i] = pubkey[i]
    M[i , i] = pubkey[0]
M[0,0] = 2**rbit

p0 = M.LLL()[0,0] // 2**rbit
q = pubkey[0] // p0
print(q)
print(long_to_bytes(int(''.join(str(int(j)) for j in [(i%q)%2 for i in c]),2)))
```

## 方法总结

本题核心是把简化同态加密还原成 AGCD 问题。公钥不是随机大数，而是一组共享隐藏因子 $q$ 的近似倍数；只要用格规约恢复 $q$，每个密文位都能通过 $(c\bmod q)\bmod 2$ 解出。

识别信号是源码中 `pk.append(p*sk + 2*r)`、明文逐 bit 加密、扰动位数明显小于私钥位数、输出了大量公钥样本。

复现时选取若干公钥构造 LLL 矩阵，先恢复与 `pubkey[0]` 相关的商或短向量，再得到 $q$。随后对 `cipher.txt` 中每个整数计算 $(i\bmod q)\bmod 2$，拼成二进制串并转字节。

参考资料要点：Approximate GCD 资料说明样本 $x_i=q_i p+r_i$ 在噪声较小时可用格短向量思路攻击，并给出了同态加密玩具构造与解密形式。原文见 https://martinralbrecht.wordpress.com/2020/03/21/the-approximate-gcd-problem/。
