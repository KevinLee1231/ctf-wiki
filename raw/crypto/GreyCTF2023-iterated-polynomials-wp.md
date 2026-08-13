# GreyCTF 2023 Iterated Polynomials

## 题目简述

题目在商环 $S=\mathbb{F}_p[y]/(a(y))$ 中从 $g=y$ 出发，反复执行代换 $y\mapsto y^2+1$。迭代次数是 32 字节 flag 正文按十六进制解释后的大整数，只给出最终多项式。虽然写成多项式复合，这个代换在 16 维系数空间上其实是线性变换。

## 解题过程

以 $1,y,\ldots,y^{15}$ 为基。对每个基向量计算

$T(y^n)=(y^2+1)^n\bmod a(y)$，

把系数排成矩阵 $M$。题目输出就是

$h=T^N(g)=M^N g$，

其中 $N$ 是待恢复的 flag 整数。

矩阵的特征值不一定都在 $\mathbb{F}_p$ 中，因此官方解法把矩阵扩展到 $\mathbb{F}_{p^{12}}$ 后对角化。若左特征分解给出特征值 $\lambda_i$，并把 $g,h$ 投影到同一特征基，则每个非零分量满足

$h_i=g_i\lambda_i^N$。

于是可分别计算：

```sage
order_i = lambda_i.multiplicative_order()
n_i = (h_i / g_i).log(lambda_i)
```

这给出 $N\equiv n_i\pmod{\operatorname{ord}(\lambda_i)}$。跳过重复阶并收集足够多的独立同余式，使用 CRT 合并得到唯一的 $N$：

```sage
N = crt(residues, orders)
body = bytes.fromhex(hex(N)[2:]).decode()
```

恢复结果为：

```text
grey{7h3_FunC710N_15_4c7U4Lly_l1N34r!}
```

## 方法总结

“反复代换多项式”看似非线性，但对固定商环中的系数向量而言，代换算子保持加法与标量乘法，因此可以矩阵化。对角化把巨大矩阵幂拆成多个有限域乘法群中的离散对数，再用 CRT 拼回指数；关键是选择能容纳特征值的扩域并处理重复阶。
