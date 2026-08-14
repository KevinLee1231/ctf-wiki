# RSA-1

## 题目简述

题目使用 $N=pq$ 和 $e=65537$ 加密 Flag，但额外在大素数域 $\mathbb{F}_{PP}$ 中泄露了关于 $p,q$ 的两条多项式关系：

$$
f(p,q)=31p^3q+50q^2+89p-1000,
$$

$$
g(p,q)=97q^5p^2+71p^2+27p+31q+3131.
$$

已知两式在模 $PP$ 下的值，可以通过消元求出 $p,q$，进而分解 RSA 模数。

## 解题过程

把泄露值移到等式左侧，得到在 $\mathbb{F}_{PP}$ 上同时为零的多项式 $F(x,y)$ 和 $G(x,y)$。分别对 $x$、$y$ 求结式可消去一个未知量：

```sage
R.<x, y> = PolynomialRing(ZZ)
F = 31*x^3*y + 50*y^2 + 89*x - 1000 - leak_f
G = 97*y^5*x^2 + 71*x^2 + 27*x + 31*y + 3131 - leak_g

q_roots = F.resultant(G, x).change_ring(GF(PP)).univariate_polynomial().roots()
p_roots = F.resultant(G, y).change_ring(GF(PP)).univariate_polynomial().roots()
```

结式的根可能包含额外候选，因此枚举根对，检查 $p\cdot q=N$，然后按标准 RSA 解密：

```python
for p0, _ in p_roots:
    for q0, _ in q_roots:
        p, q = int(p0), int(q0)
        if p * q != N:
            continue
        phi = (p - 1) * (q - 1)
        d = pow(0x10001, -1, phi)
        print(long_to_bytes(pow(c, d, N)))
```

得到：

```text
greyhats{solving_equations_in_field_is_not_hard_qZCRZdsw79Yy3dde}
```

## 方法总结

- 核心技巧：在有限域中用多项式结式消元，恢复 RSA 的两个素因子。
- 识别信号：除了 $N,c$ 外还给出多个关于 $p,q$ 的独立代数关系。
- 复用要点：结式只负责产生候选，必须将根代回原式并检查 $pq=N$，避免采用伪根。
