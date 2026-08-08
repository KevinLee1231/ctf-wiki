# MiniLCTF2022 Double SS revenge Writeup

## 题目简述

revenge 版本修补了整数环上的非预期解：多项式求值改在公开素数 $p$ 的有限域 $\mathbb F_p$ 中进行，但仍只给出 31 个点，次数上界仍为 31。首个 4 字节系数以 `#` 开头，其余三字节来自给定字符表，因此仍可用格式约束确定唯一自由度。

## 解题过程

在 $R=\mathbb F_p[x]$ 中对 31 组数据插值得到 $g(x)$，并构造消失多项式

$$
h(x)=\prod_i(x-x_i).
$$

全部候选仍满足 $f=g+t h$，区别只是 $t\in\mathbb F_p$。枚举首系数的后三个字符后，用常数项求出

$$
t=(a_0-g(0))h(0)^{-1}\pmod p.
$$

然后恢复 $f$ 的 32 个系数。flag 所在的前 16 个系数来自 4 字节块，所以其标准整数代表必须小于 $2^{32}$；拼接后还必须出现正确的 `#...#` 边界。SageMath 中的核心步骤为：

```sage
F = GF(p)
R.<x> = PolynomialRing(F)
g = R.lagrange_polynomial([(F(xi), F(ci)) for xi, ci in samples])
h = prod(x - F(xi) for xi, _ in samples)

for suffix in product(table, repeat=3):
    a0 = F(bytes_to_long(b"#" + bytes(suffix)))
    t = (a0 - g[0]) / h[0]
    f = g + t * h
    coeffs = [int(f[i]) for i in range(32)]
    if all(v < 2**32 for v in coeffs[:16]):
        test(bytes_from_coeffs(coeffs[:16]))
```

最终唯一解为：

```text
miniLCTF{ZZ_s33ms_s0meth1ng_Wr0ng!_Wh@t_about_mod_f1eld?}
```

## 方法总结

有限域修复了利用整数位数直接读系数的捷径，却没有消除“少一条方程、一个自由度”的结构。正确做法是把线性解空间显式参数化，再用明文格式约束确定参数。计算时必须始终在 $\mathbb F_p$ 中插值和求逆；若先在有理数域求解再对 $p$ 取模，容易因分母和表示方式产生错误。
