# L3akCTF 2025 Dumber Writeup

## 题目简述

题目在一个未知素数域上构造椭圆曲线

$$
E:y^2=x^3+ax+b\pmod p,
$$

并输出四个点 $P,U,Q,V$，其中

$$
U=m_1P,\qquad V=m_2Q.
$$

$m_1,m_2$ 分别是 flag 前后两段转成的大整数。题目没有给出 $p,a,b$，看似连曲线都无法建立；但四个点足以恢复曲线参数，而恢复出的曲线又是可被 Smart Attack 直接求离散对数的 anomalous curve。

仓库仅提供题目与输出，没有官方求解脚本。下面的解法根据题目源码、公开输出及原作者题解思路整理。

## 解题过程

### 由四个点恢复模数

对任一点 $(x_i,y_i)$，令

$$
T_i=y_i^2-x_i^3.
$$

因为点在曲线上，所以存在整数 $k_i$ 使

$$
T_i=ax_i+b+k_ip.
$$

两点相减可消去 $b$：

$$
D_{ij}=T_i-T_j=a(x_i-x_j)+k_{ij}p.
$$

再取两组差并交叉相乘，就能消去 $a$：

$$
N=D_{ij}(x_k-x_l)-D_{kl}(x_i-x_j).
$$

$N$ 必然是 $p$ 的倍数。用多组不同下标构造 $N$，其最大公约数通常为 $p$ 或 $p$ 乘上一个很小的偶然因子：

```python
from itertools import combinations
from math import gcd

points = [P, U, Q, V]
T = [y * y - x * x * x for x, y in points]

multiples = []
for (i, j), (k, l) in combinations(combinations(range(4), 2), 2):
    n = (T[i] - T[j]) * (points[k][0] - points[l][0])
    n -= (T[k] - T[l]) * (points[i][0] - points[j][0])
    if n:
        multiples.append(abs(n))

candidate = 0
for n in multiples:
    candidate = gcd(candidate, n)
```

若最大公约数还带有小因子，就逐个剥离，并以“候选数为素数且四个点都满足同一条非奇异曲线”为准筛选真正的 $p$。

### 恢复 $a,b$ 并验证曲线

已知 $p$ 后，任取横坐标不同的两点，有

$$
a=\frac{(y_1^2-y_2^2)-(x_1^3-x_2^3)}
{x_1-x_2}\pmod p,
$$

$$
b=y_1^2-x_1^3-ax_1\pmod p.
$$

在 Sage 中重建曲线并验证四个点：

```python
Fp = GF(p)
a = ((y1^2 - y2^2) - (x1^3 - x2^3)) / (x1 - x2)
b = y1^2 - x1^3 - a * x1
E = EllipticCurve(Fp, [a, b])

assert all(E(x, y) in E for x, y in points)
```

随后计算曲线阶，得到

$$
\#E(\mathbb F_p)=p.
$$

这说明曲线的 Frobenius trace 为 1，是 anomalous curve。此时一般椭圆曲线离散对数的安全性不再成立，可以使用 Smart Attack。

### Smart Attack 求两个标量

Smart Attack 将曲线和点提升到 $p$ 进数域 $\mathbb Q_p$。对 anomalous curve，点乘 $p$ 后会进入形式群；利用局部参数

$$
\phi(R)=-\frac{x([p]R)}{y([p]R)}
$$

可以把椭圆曲线上的标量乘法转成模 $p$ 的线性关系。若 $U=mP$，则

$$
m=\frac{\phi(U)}{\phi(P)}\pmod p.
$$

一个精简的 Sage 实现如下：

```python
def smart_attack(P, Q):
    E = P.curve()
    p = E.base_ring().order()
    K = Qp(p, 2)

    for _ in range(16):
        a1 = ZZ(E.a1()) + randint(0, p - 1) * p
        a2 = ZZ(E.a2()) + randint(0, p - 1) * p
        a3 = ZZ(E.a3()) + randint(0, p - 1) * p
        a4 = ZZ(E.a4()) + randint(0, p - 1) * p
        a6 = ZZ(E.a6()) + randint(0, p - 1) * p
        Ep = EllipticCurve(K, [a1, a2, a3, a4, a6])

        P_candidates = Ep.lift_x(K(ZZ(P[0])), all=True)
        Q_candidates = Ep.lift_x(K(ZZ(Q[0])), all=True)
        P_candidates = [R for R in P_candidates
                        if ZZ(R[1]) % p == ZZ(P[1])]
        Q_candidates = [R for R in Q_candidates
                        if ZZ(R[1]) % p == ZZ(Q[1])]
        if not P_candidates or not Q_candidates:
            continue

        Pl = p * P_candidates[0]
        for Ql0 in Q_candidates:
            Ql = p * Ql0
            phi_p = -Pl[0] / Pl[1]
            phi_q = -Ql[0] / Ql[1]
            m = ZZ(phi_q / phi_p) % p
            if m * P == Q:
                return m

    raise ValueError("lift failed")
```

分别计算

```python
m1 = smart_attack(P, U)
m2 = smart_attack(Q, V)
flag = long_to_bytes(m1) + long_to_bytes(m2)
print(flag)
```

得到：

```text
L3AK{5m4rt1_1n_Th3_h00000d!!!}
```

仓库缺少官方 solver 时，曲线参数恢复与 Smart Attack 的完整推导可参考原作者公开的 [Dumber 题解](https://k3sero.github.io/posts/Dumber-L3AK2025/)。该链接中的关键步骤已经在本文中展开，不依赖外链也能复现。

## 方法总结

未知椭圆曲线参数不等于信息不足。多个公开点会给出同余方程，先用差分和交叉乘积把 $a,b$ 消去，就能从若干个 $p$ 的倍数中取最大公约数恢复模数。重建曲线后还要检查曲线阶；本题决定性漏洞是 $\#E(\mathbb F_p)=p$，因此可用 Smart Attack 把 ECDLP 降为 $p$ 进数形式群中的线性计算。
