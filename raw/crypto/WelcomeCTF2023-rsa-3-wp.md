# RSA-3

## 题目简述

题目使用 512 位素数 $p,q$ 组成 RSA 模数 $N=pq$，另生成约 512 位秘密 $k$。它在四个约 750 位素数 $s_i$ 下泄露四条关于 $p,k$ 的二元多项式同余，例如：

$$
17p^2+27k^2\equiv v_0\pmod{s_0},
$$

其余三式分别包含一次项、二次项和 $pk$。单条同余不足以确定未知量，但四条关系可先通过 CRT 合为模 $S=\prod s_i$ 的多项式，再使用多元 Coppersmith 小根方法恢复 $p,k$。

## 解题过程

先在 $\mathbb{Z}[x,y]$ 中建立四个“多项式减泄露值”，再对每个系数做 CRT：

```sage
R.<x, y> = PolynomialRing(ZZ)
polys = [
    17*x^2 + 27*y^2 - v0,
    31*x + 71*y + 107*x*y - v1,
    x^2 + y^2 + x + y + x*y - v2,
    11*x^2 + 51*x*y + 13*y - v3,
]

S = prod(moduli)
def crt_poly(values, mods):
    total = prod(mods)
    return sum(
        value * (total // mod) * inverse_mod(total // mod, mod)
        for value, mod in zip(values, mods)
    )

F = crt_poly(polys, moduli).change_ring(Zmod(S))
```

真实根满足 $F(p,k)=0\pmod S$，且边界约为 $p<2^{513}$、$k<2^{513}$。官方脚本构造移位多项式族 $S^{m-i}F^i x^a y^b$，按根边界缩放系数矩阵后做 LLL，得到在整数域上同样以 $(p,k)$ 为根的短多项式；当这些多项式生成零维理想时，用 `variety` 枚举小根。

```sage
roots = small_roots(F, bounds=[2^513, 2^513], m=5, d=5)
for p0, k0 in roots:
    p0 = int(p0)
    if N % p0:
        continue
    q0 = N // p0
    phi = (p0 - 1) * (q0 - 1)
    d = inverse_mod(0x10001, phi)
    print(long_to_bytes(power_mod(c, d, N)))
```

以 $p\mid N$ 作为强校验，最终解得：

```text
greyhats{mastering_the_art_of_equation_solving_7qtPbU5TshVDJrMp}
```

## 方法总结

- 核心技巧：CRT 合并多个模数下的多项式泄露，再用多元 Coppersmith/LLL 求有界小根。
- 识别信号：未知 RSA 因子同时出现在多条不同模数的低次数同余中，且未知量位数明显小于模数乘积提供的约束强度。
- 复用要点：LLL 返回的是候选关系，最终必须用原同余和 $p\mid N$ 验证；根边界、移位次数和缩放决定攻击是否成功。
