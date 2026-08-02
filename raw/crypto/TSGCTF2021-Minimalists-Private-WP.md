# TSGCTF2021 Minimalist's Private WP

## 题目简述

题目给出 RSA 公钥 $(N,e)$ 和密文 $c$。自定义实现没有直接要求 $L=\varphi(N)$，只验证随机底数满足 $a^L\equiv1\pmod N$、$ed\equiv1\pmod L$，并施加“小私钥参数”约束：

```python
assert self.e * self.d % self.L == 1
assert self.L * self.L <= 10000 * self.N
```

生成器令两个素数具有很大的公共因子：

$$
p=ig+1,\qquad q=jg+1.
$$

其中 $\gcd(i,j)=1$ 且 $ij\le10000$。因此 Carmichael 指数为：

$$L=\operatorname{lcm}(p-1,q-1)=ijg$$

目标是利用很小的 $i,j$ 从 $N$ 恢复 $p,q$。

## 解题过程

展开模数：

$$
N=(ig+1)(jg+1)
=ijg^2+(i+j)g+1
$$

所以 $g$ 满足二次方程：

$$ijg^2+(i+j)g-(N-1)=0$$

其判别式为：

$$D=(i+j)^2+4ij(N-1)$$

若候选 $(i,j)$ 正确，$D$ 必须是完全平方数，而且：

$$g=\frac{\sqrt D-i-j}{2ij}$$

由于 $ij\le10000$，直接枚举 $1\le i\le j$ 以及所有满足范围的 $j$ 即可。实现时必须用整数平方根并验证整除关系与最终乘积：

```python
from math import isqrt

for i in range(1, 101):
    for j in range(i, 10000 // i + 1):
        D = (i + j) ** 2 + 4 * (N - 1) * i * j
        root = isqrt(D)
        if root * root != D:
            continue
        if (root - i - j) % (2 * i * j) != 0:
            continue

        g = (root - i - j) // (2 * i * j)
        p, q = i * g + 1, j * g + 1
        if p > 1 and q > 1 and p * q == N:
            break
```

本实例在 $i=13$ 时命中正确结构。得到 $p,q$ 后，按普通 RSA 计算：

$$
\varphi(N)=(p-1)(q-1),\qquad d=e^{-1}\bmod\varphi(N).
$$

再解密 $m=c^d\bmod N$ 并转成大端字节。仓库 solver 已在本地运行验证，输出：

```text
TSGCTF{Roll_Safe:_You_c4n't_be_exploited_1f_you_are_a_minimali5t_enough_and_y0u_don't_have_any_s3crets_in_your_mind}
```

## 方法总结

本题泄露的不是传统意义上的“小私钥 $d$”，而是过小的群指数 $L$。$L^2\le10000N$ 对应 $p-1$ 与 $q-1$ 含有巨大的公共因子，使两个素数能写成 $ig+1$ 与 $jg+1$，且小系数乘积可穷举。把 $N$ 展开为关于 $g$ 的二次方程后，完全平方判别式就成了高效筛选条件。RSA 参数生成必须让 $p-1,q-1$ 的结构不可预测，不能只检查若干随机底数满足指数关系就认为密钥安全。
