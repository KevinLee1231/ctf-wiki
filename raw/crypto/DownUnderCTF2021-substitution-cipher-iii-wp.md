# DownUnderCTF 2021 - Substitution Cipher III

## 题目简述

这不是普通单表替换，而是一个 Matsumoto–Imai 风格的多变量公钥密码。参数为 $q=2,n=80$；每段 10 字节明文被编码成 $\mathbb F_2^{80}$。私钥由两个可逆仿射变换 $S,T$ 和扩域 $E=\mathbb F_{2^{80}}$ 上的中心映射 $P(a)=ra^{q+1}=ra^3$ 组成，公开密钥是它们复合后得到的 80 个二次多项式。

flag 去掉 `DUCTF{}` 后被拆成两个 10 字节块分别加密。题目给出公开多项式和两个 80 位密文，目标是在不知道 $S,T,r$ 的情况下恢复明文。

## 解题过程

Patarin 对 Matsumoto–Imai 结构的攻击不需要恢复私钥。令 $x$ 为明文向量、$y$ 为密文向量，并定义：

$$
a=\varphi(S(x)),\qquad b=\varphi(T^{-1}(y)).
$$

中心映射给出 $b=ra^{q+1}$。两边取适当幂并整理可得：

$$
ab^q=r^{q-1}a^{q^2}b.
$$

在特征 2 的扩域中，Frobenius 映射 $v\mapsto v^q$ 和 $v\mapsto v^{q^2}$ 都是 $\mathbb F_2$ 线性的；$a$、$b$ 又分别是 $x$、$y$ 的仿射函数。因此上式的每个坐标都能改写成对 $x,y$ 双线性、分别至多一次的关系：

$$
\sum_{i,j}\gamma_{ij}x_i y_j
+\sum_i\alpha_i x_i
+\sum_j\beta_j y_j
+\delta=0.
$$

这些未知系数共有 $n^2+2n+1=(n+1)^2=6561$ 个。公开密钥允许自行选择明文并计算密文，因此生成至少 6561 组 $(X,Y)$，每组构造一行：

```sage
def relation_row(pair):
    X, Y = pair
    row = [F(X[i] * Y[j]) for i in range(n) for j in range(n)]
    row += [F(bit) for bit in X]
    row += [F(bit) for bit in Y]
    row += [F(1)]
    return row

pairs = [(pt, tuple(p(*pt) for p in pubkey))
         for pt in chosen_sparse_plaintexts]
M = Matrix(GF(2), [relation_row(pair) for pair in pairs])
relations = M.right_kernel()
```

选择稀疏明文向量只是在加速公开二次多项式求值，不改变攻击原理。`M` 的右核给出对所有合法明密文对成立的系数关系。

对目标密文 $Y=C$，把 $y_j$ 固定为已知位。每个核向量随即变成关于 $x_0,\ldots,x_{79}$ 的线性方程：

```sage
def equations_for_ciphertext(relations, C):
    X = Kb.gens()
    Y = list(map(int, C))
    equations = []

    for coeff in relations.basis():
        eq = sum(coeff[n*i + j] * X[i] * Y[j]
                 for i in range(n) for j in range(n))
        eq += sum(coeff[n^2 + i] * X[i] for i in range(n))
        eq += sum(coeff[n^2 + n + j] * Y[j] for j in range(n))
        eq += coeff[n^2 + 2*n]
        equations.append(G(eq))
    return Sequence(equations)
```

求这个线性系统的解，去掉齐次化常数位，将 80 个比特还原为 10 字节；若核中存在多个候选，可用可打印字符约束选择正确解。分别处理两个密文并重新加上外层格式：

```text
DUCTF{MQ_1s_fun_a5e39cf21a}
```

## 方法总结

多变量二次方程一般难解，并不代表任何由二次多项式组成的公钥都安全。Matsumoto–Imai 中隐藏的扩域幂映射保留 Frobenius 线性结构，使任意明密文对都满足可学习的双线性关系。识别信号是“仿射隐藏 + 扩域中心幂映射 + 巨大的多变量公钥”；有公开加密能力时，可用足量选择明文样本通过线性代数恢复关系，再把固定密文下的问题降为线性方程组。
