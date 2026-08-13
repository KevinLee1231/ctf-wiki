# GreyCTF2023 QRSA

## 题目简述

附件 PDF 定义了二次整数环上的 RSA。flag 被平分为整数 $M_a,M_b$，组成

$M=M_a+M_b\sqrt D$，其中 $D=41$。

同样地，$p=p_a+p_b\sqrt D$、$q=q_a+q_b\sqrt D$，公开模数与密文为

$N=pq=N_a+N_b\sqrt D$，

$C\equiv M^e\pmod N=C_a+C_b\sqrt D$。

同余的含义是 $x-y=kN$，其中 $k$ 也属于该二次整数环。PDF 还说明两段明文长度接近且均为可打印 ASCII；这些信息均已转写为上述代数条件，不需要保留纯公式截图。

## 解题过程

对 $u=a+b\sqrt{41}$ 定义共轭 $\bar u=a-b\sqrt{41}$，范数为

$\operatorname{Norm}(u)=u\bar u=a^2-41b^2$。

因此先计算并分解公开模数的整数范数：

```sage
norm_N = N_a^2 - 41*N_b^2
fac = factor(norm_N)
```

范数把二次环中的模数结构投影成普通整数分解。根据这些素因子构造候选群指数；官方脚本分别使用

```sage
ord1 = product((r-1)^2 * (r^2-1) * r^e for r, e in fac)
ord2 = product((r-1) * r^(e-1) for r, e in fac)
d1 = inverse_mod(65537, ord1)
d2 = inverse_mod(65537, ord2)
```

实现二次整数的乘法、共轭除法取整和模约减后，分别计算 $C^{d_1}\bmod N$ 与 $C^{d_2}\bmod N$。把候选结果的两个系数按大端序拼接，并用 PDF 给出的“长度接近、可打印”条件筛选：

```python
candidate = long_to_bytes(m.a) + long_to_bytes(m.b)
```

唯一合理结果为：

```text
grey{x3VkGD3K2SK5s4JW_Lmao_why_do_RSA_in_quadratic_integer}
```

## 方法总结

本题不是把普通 RSA 参数换个记号，而是把运算搬到 $\mathbb{Z}[\sqrt{41}]$。突破口是范数具有乘法性：$\operatorname{Norm}(pq)=\operatorname{Norm}(p)\operatorname{Norm}(q)$，从而把环上因式结构转回可分解的整数。实现自定义环 RSA 时，还必须严格复现题目定义的模约减，并用明文格式约束区分不同群指数产生的候选解。
