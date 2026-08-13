# GreyCTF2022 - Polynomial

## 题目简述

题目在有限域多项式商环中做指数运算，并把共享元素用于解密。直接在巨大商环中枚举指数不可行，但“乘以固定多项式”是有限维向量空间上的线性变换，可用矩阵特征值把问题拆成较小有限域中的离散对数。

## 解题过程

设商环为 $R=\mathbb F_p[x]/(F(x))$，固定元素为 $g(x)$。在基 $1,x,\ldots,x^{d-1}$ 下，乘法映射

$$T_g:h\mapsto g h\bmod F$$

对应一个 $d\times d$ 矩阵。公开的 $g^a$ 则对应 $T_g^a$。分解 $F$ 的不可约因子，或对伴随/乘法矩阵求特征值，可在各扩域分量中得到若干标量关系 $\lambda_A=\lambda_g^a$。

```sage
factors = F.factor()
for fi, _ in factors:
    K.<u> = GF(p**fi.degree(), modulus=fi)
    residues.append(discrete_log(K(A), K(g)))
    moduli.append(K(g).multiplicative_order())
a = CRT_list(residues, moduli)
```

用恢复的指数计算另一公开元素的 $a$ 次幂，按源码派生密钥并解密。官方解法得到：

```text
grey{58693e0cdfc89ac44579cfd6343ee2854afba49c}
```

## 方法总结

有限维代数中的幂运算经常可以通过正则表示线性化。真正需要核对的是每个分量中生成元的实际阶，而不是机械使用 $p^d-1$；各离散对数同余应在兼容条件下合并，并用原商环重算公开值验证指数。
