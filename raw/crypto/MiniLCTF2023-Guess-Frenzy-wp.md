# MiniLCTF2023 - Guess Frenzy

## 题目简述

服务生成 52 个小于 50 位素数 $q$ 的素数 $p_i$，随机选择其中至少一半，返回

$$
r=\prod_i p_i^{x_i}\pmod q,\qquad x_i\in\{0,1\}.
$$

选手需要把选择向量按源码的高位在前顺序编码成整数 `key`。这是模乘形式的子集积问题；由于 $q$ 只有 50 位，可以先在 $GF(q)^*$ 中取离散对数，把它转成模 $q-1$ 的低密度背包，再用 BKZ 找短向量。

## 解题过程

取 $GF(q)^*$ 的生成元 $g$，令

$$
m_i=\log_g p_i,\qquad s=\log_g r.
$$

原式变为

$$
\sum_i x_i m_i\equiv s\pmod{q-1}.
$$

构造 $n+2$ 维格：第一行放 $C(q-1)$，随后每行放 $Cm_i$ 和一个对角元 2，最后一行放 $Cs$、$n$ 个 1 以及末尾的 1。放大常数 $C$ 迫使短向量第一坐标为 0；目标向量其余选择坐标为 $\pm1$，最后一维为 $\pm1$。本题规模下 `block_size=30` 通常足够。

```python
from functools import reduce
from operator import mul
from sage.all import GF, Matrix, ZZ


def solve_subset(primes, q, product):
    field = GF(q)
    g = field.primitive_element()
    logs = [int(field(p).log(g)) for p in primes]
    target = int(field(product).log(g))

    n = len(primes)
    scale = 1 << 80
    rows = [[scale * (q - 1)] + [0] * (n + 1)]
    for i, value in enumerate(logs):
        row = [0] * (n + 2)
        row[0] = scale * value
        row[i + 1] = 2
        rows.append(row)
    rows.append([scale * target] + [1] * (n + 1))

    reduced = Matrix(ZZ, rows).BKZ(block_size=30)
    for v in reduced.rows():
        for sign in (1, -1):
            w = sign * v
            if w[0] != 0 or w[-1] != 1:
                continue
            bits = [(1 - int(w[i])) // 2 for i in range(1, n + 1)]
            if any(bit not in (0, 1) for bit in bits):
                continue
            check = 1
            for p, bit in zip(primes, bits):
                if bit:
                    check = check * p % q
            if check == product:
                return int("".join(map(str, bits)), 2)
    raise ValueError("BKZ did not expose the target vector; retry the instance")
```

把返回的整数提交给服务即可得到：

```text
miniL{C0ngr4tu1atio5!U_D0_KnOw_b4ckpacK!!!}
```

## 方法总结

模乘子集问题可以通过离散对数变为模加法背包，但前提是群足够小，离散对数本身可算。本题的关键是优化格嵌入：用 2 把布尔变量映射到 $\pm1$，再放大同余列以突出目标短向量。BKZ 具有概率性，必须对候选重新计算原始模乘积，不能仅凭向量形状认定答案。
