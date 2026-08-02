# TSGCTF2020 Sweet like Apple Pie WP

## 题目简述

题目把 flag 解释成小于 $2^{500}$ 的整数 $m$，用 300 位十进制精度计算一个自定义正弦函数并公开结果：

```python
def sin(x):
    x = Decimal(x) % pi()
    p, factor = 0, x
    for n in range(10000):
        p += factor
        factor *= -(x ** 2) / ((2 * n + 2) * (2 * n + 3))
    return p
```

函数先对 $\pi$ 而非 $2\pi$ 取模，再用泰勒级数求正弦。直接反正弦只能得到余数，整数 $m$ 还包含一个未知的巨大 $\pi$ 倍数；解题关键是利用 $\pi$ 的连分数收敛分数，把近似实数同余转换成整数模方程。

## 解题过程

设公开值为 $x$。在区间 $[0,\pi)$ 内，满足 $\sin r=x$ 的候选余数有两个：

$$
r_1=\arcsin x,
\qquad
r_2=\pi-\arcsin x.
$$

真实 flag 整数满足 $m\equiv r_i\pmod\pi$。取 $\pi$ 的一个高精度连分数收敛分数：

$$
\pi\approx\frac{p}{q}.
$$

两边乘以 $q$ 后，模 $\pi$ 的关系近似变为：

$$
qm\equiv qr_i\pmod p.
$$

当收敛分数足够精确时，$qr_i$ 极接近整数。令 $n=\operatorname{round}(qr_i)$，即可在整数环中求：

$$
m\equiv n q^{-1}\pmod p.
$$

官方 solver 逐个生成 $\pi$ 的收敛分数，并对两个反正弦分支尝试候选：

```python
for residue in [arcsin(x), PI - arcsin(x)]:
    pp, pq, p, q = 0, 1, 1, 0
    tail = PI

    for _ in range(500):
        a = int(tail)
        pp, p = p, a * p + pp
        pq, q = q, a * q + pq
        tail = Decimal(1) / (tail - a)

        n = round(residue * Decimal(q))
        candidate = n * pow(q, -1, p) % p

        if candidate < 2 ** 500:
            if abs(sin(candidate) - x) < Decimal("1e-290"):
                print(candidate.to_bytes(
                    (candidate.bit_length() + 7) // 8, "big"
                ))
                break
```

必须把候选重新代入题目原函数，因为有限精度、错误反正弦分支和较差的早期收敛分数都可能产生伪解。本地运行官方 solver 后得到：

```text
TSGCTF{Tteugeopgo_Dara_Sweee-ee-ee-ee-eet_1ik3_4pple_Pie_Pie}
```

## 方法总结

本题泄漏的是整数对无理数 $\pi$ 取模后的高精度结果。连分数提供了 $\pi\approx p/q$ 的最佳有理逼近，使实数模关系在乘以 $q$ 后变成可解的整数同余；反正弦的两个分支则必须同时枚举。处理高精度三角函数密码题时，应先检查周期归约是否正确，再把未知整数、周期和观测值写成近似线性关系，最后以原始高精度函数严格回代。
