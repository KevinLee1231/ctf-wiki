# MiniLCTF2022 Double SS Writeup

## 题目简述

本题沿用 Double S 的 31 次整数多项式，但只公开 31 组点值，因此系数向量还剩一个自由度。flag 内容仍被包在 `#` 之间，每 4 字节转换为一个系数；补位字符来自给定字符表。首个系数对应 `#` 与 flag 的前三个字符，这一格式约束正好补足缺失的一条方程。

## 解题过程

设公开点为 $(x_i,c_i)$，先插值得到一个满足全部 31 个点的次数不超过 30 的多项式 $g(x)$，再令

$$
h(x)=\prod_{i=0}^{30}(x-x_i).
$$

所有次数不超过 31、且通过这些点的多项式都可写为

$$
f(x)=g(x)+t\,h(x).
$$

未知量只剩整数 $t$。首个 4 字节系数 $a_0$ 的最高字节必为 `#`，后三字节来自题目字符表。枚举这三个字符便得到候选 $a_0$，并由

$$
t=\frac{a_0-g(0)}{h(0)}
$$

求出 $t$。只保留 $t$ 为整数、前 16 个系数均落在 $[0,2^{32})$、且拼接结果满足 `#...#` 格式的候选。伪代码如下：

```python
g = interpolate_over_QQ(samples)
h = prod(x - xi for xi, _ in samples)

for c1 in table:
    for c2 in table:
        for c3 in table:
            a0 = bytes_to_long(bytes([ord("#"), c1, c2, c3]))
            numerator = a0 - g[0]
            if numerator % h[0]:
                continue
            t = numerator // h[0]
            coeffs = coefficients(g + t * h, length=32)
            if all(0 <= v < 2**32 for v in coeffs[:16]):
                check_candidate(coeffs)
```

附件数据只产生一个合法候选：

```text
miniLCTF{SS5S_c0u1d_b3_brUt3-f0rc3_1f_th3_c0eff_1s_sm4ll}
```

## 方法总结

少一个插值点并不意味着必须暴力搜索整个多项式。31 个点把 32 维系数空间压缩成一条仿射直线，格式信息只需确定这条直线上的参数 $t$。把解写成 $g+t h$，再用已知前缀、字符表、32 位系数范围和分隔符联合筛选，比直接对全部系数建模更清晰，也更容易验证唯一性。
