# Common

## 题目简述

题目是共模 RSA 小私钥攻击的变体。第一阶段给出素数高位信息，可用 Coppersmith `small_roots` 恢复提示；提示指向“多个解密指数下扩展 Wiener 攻击”和 common modulus small private exponent 论文。第二阶段利用 Wiener equations 与 Guo equations 组合出短向量问题，通过 LLL 求出包含 $k_i,d_i,g$ 关系的向量，恢复 $\varphi(N)$，再分解 $N$ 解密密文。

题目资料地址：https://gist.github.com/LurkNoi/dfe86ed4d16776242251318b380336e7

题目生成脚本约束 `len(hint) == 28`，随机生成 1024-bit `p,q`，令 `N=p*q`，并用 `lcm(p-1,q-1)` 作为 Carmichael 模数。两个私钥指数 `d1,d2` 是 700-bit prime，`e1,e2` 分别为它们在 `lam` 下的逆元；最终密文仍使用标准 `65537` 加密 flag。这里的 `700/2048` 正好对应正文格构造里使用的 $\delta$ 规模。

## 解题过程

1. 简单的 `Factoring with High Bits Known`, sage 构造对应 polynomial 使用自带的 `small_roots` 可以解出 `hint = '3-540-46701-7_14 or 2009/037'`
2. 提示对应两类共模小私钥攻击资料：\[[1](https://link.springer.com/chapter/10.1007/3-540-46701-7_14 "Extending Wiener’s Attack in the Presence of Many Decrypting Exponents")\] 讨论多个解密指数存在时如何把 Wiener 攻击扩展成格问题；\[[2](https://eprint.iacr.org/2009/037 "Common Modulus Attacks on Small Private Exponent RSA and Some Fast Variants (in Practice)")\] 复查 common modulus small private exponent 场景，重点给出 Guo continued-fraction attack 和 Howgrave-Graham/Seifert lattice attack 的实践边界。本题只有两个 `e_i,d_i` 共享同一个 `N`，且 `d_i` 明显小于 `N`，所以按论文里的 `k_i,d_i,g` 关系构造低维格，而不是做普通共模明文攻击。

记 Wiener equations 和 Guo equations 为

$W_i: e_i d_i g - k_i N = g - k_i s$

$G_{i,j}: k_i d_j e_j - k_j d_i e_i = k_i - k_j$

则 $k_1 k_2, k_2 W_1, g G_{1,2}, W_1 W_2$ 转化成矩阵形式, 有 $\mathbf{x} B = \mathbf{v}$, 其中

$$\mathbf{x} = (k_1 k_2,\ k_2 d_1 g,\ k_1 d_2 g,\ d_1 d_2 g^2)$$  
$$B = \begin{bmatrix}
1 & -N & 0 & N^2 \\
& e_1 & -e_1 & -e_1 N \\
& & e_2 & -e_2 N \\
& & & e_1 e_2
\end{bmatrix}$$  
$$\mathbf{v} = (k_1 k_2,\ k_2(g - k_1 s),\ g(k_1 - k_2),\ (g - k_1 s)(g - k_2 s))$$

令 $D = \text{diag}(N, N^{1/2}, N^{1+\delta}, 1)$, 使其满足 Minkowski’s bound, 有 $\|\mathbf{v}D\| < \text{vol}(L) = |\det(B) \det(D)|$  
即 $N^{2(1/2+\delta)} < 2N^{(13/2+\delta)/4}$, $\delta < 5/14 - \epsilon$.

1. 利用 `LLL` 求出最短向量 $\mathbf{v}D$, 进而求出 $\mathbf{x}$, 根据 Wiener’s attack, $\varphi(N) = g(ed-1)/k = \lfloor edg/k \rfloor$
2. 有了 $\varphi(N)$ 即可构造一元二次方程分解 $N$.
```python
#!/usr/bin/sage -python
from sage.all import *
from Crypto.Util.number import long_to_bytes
import gmpy2

_p = p0 - (p0&(2**668-2**444))
PR = PolynomialRing(Zmod(N1), 'x')
x = PR.gen()
f = 2**444 * x + _p
f = f.monic()
r = f.small_roots(X=2**224, beta=0.5)
p1 = ZZ(_p + r[0]*2**444)
hint = ( p1.__xor__(p0) )>>444
print long_to_bytes(hint)

alpha2 = 700./2048
M1 = int(gmpy2.mpz(N)**0.5)
M2 = int( gmpy2.mpz(N)**(1+alpha2) )
D = diagonal_matrix(ZZ, [N, M1, M2, 1])
B = Matrix(ZZ, [ [1, -N,   0,  N**2],
                 [0, e1, -e1, -e1*N],
                 [0,  0,  e2, -e2*N],
                 [0,  0,   0, e1*e2] ]) * D

L = B.LLL()

v = Matrix(ZZ, L[0])
x = v * B**(-1)
phi = (x[0,1]/x[0,0]*e1).floor()

PR = PolynomialRing(ZZ, 'x')
x = PR.gen()
f = x**2 - (N-phi+1)*x + N
p, q = f.roots()[0][0], f.roots()[1][0]

d = inverse_mod( 65537, (p-1)*(q-1) )
m = power_mod(c, d, N)
print long_to_bytes(m)
```

## 方法总结

- 核心技巧：先用已知高位分解恢复论文提示，再把多个小私钥 RSA 关系转化为格上的 shortest vector problem。
- 识别信号：共模 RSA、多个公钥指数、私钥指数偏小、提示指向 Wiener/Guo 方程时，应考虑多指数扩展 Wiener 攻击而不是单独跑 common modulus 明文攻击。
- 复用要点：格构造中缩放矩阵 `D` 决定短向量是否落入 Minkowski bound；拿到短向量后从 $\lfloor edg/k \rfloor$ 恢复 $\varphi(N)$，再用二次方程分解模数。

