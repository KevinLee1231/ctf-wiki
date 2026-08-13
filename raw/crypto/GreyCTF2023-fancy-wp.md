# GreyCTF2023 Fancy

## 题目简述

题目在商环

$R=\mathbb{F}_p[x,y]/(x^3-y^2+1,\ y^7-11)$

中实现类似 Diffie-Hellman 的交换：公开 $g=1+x+y$、$A=g^a$、$B=g^b$，再用 $B^a$ 的系数派生 SHAKE-256 密钥。商环作为 $\mathbb{F}_p$ 向量空间只有 $3\times7=21$ 维，环乘法可以完全线性化。

## 解题过程

以基 $x^iy^j$（$0\le i<3,0\le j<7$）表示环元素。对任意 $u\in R$，映射 $v\mapsto uv$ 是一个 $21\times21$ 矩阵。分别构造 $M_g$ 与 $M_A$：

```sage
def hom(k):
    # 第 j 列是 k 与第 j 个基向量相乘后的系数
    return Matrix(GF(p), multiplication_columns(k))

G = hom(g)
AA = hom(A)
```

分解 $G$ 的特征多项式，并在相应扩域中选取特征向量。因为 $A=g^a$，在每个一维特征分量上都有 $\lambda_A=\lambda_g^a$。对这些小阶乘法群分别求离散对数，随后用 CRT 合并指数同余：

```sage
residues.append(TA[0, 0].log(TG[0, 0]))
moduli.append(TG[0, 0].multiplicative_order())
a = crt(residues, moduli)
shared = B^a
```

将 `shared` 的系数按题目顺序连接，输入 SHAKE-256 生成与密文等长的流并异或，得到：

```text
grey{fancy_group_sometimes_have_small_order_QCuTNsgba9myNdsp}
```

## 方法总结

“花哨”的商环并没有隐藏其有限维线性结构。把乘法写成矩阵后，幂运算会在特征分量上退化为普通有限域乘法；若各分量阶较小，离散对数便可分而治之，再用 CRT 拼回指数。分析自定义代数群时，应同时检查向量空间维数、乘法矩阵的特征分解和元素实际阶。
