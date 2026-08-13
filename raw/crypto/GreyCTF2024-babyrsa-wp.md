# babyrsa

## 题目简述

48 字节 flag 被切成三个 16 字节整数，分别使用三个相关 RSA 模数加密。题目生成三组安全素数关系 $P=2p+1$、$Q=2q+1$、$R=2r+1$，但模数交叉组合为 $N_1=pQ$、$N_2=qR$、$N_3=rP$。三个模数共同泄露了三个小写素数之间的代数关系。

## 解题过程

把 $p,q,r$ 作为整数多项式环中的未知量，公开模数给出：

$$
\begin{aligned}
p(2q+1)&=N_1,\\
q(2r+1)&=N_2,\\
r(2p+1)&=N_3.
\end{aligned}
$$

这正是三个方程、三个未知数。用 Sage 建立理想并求 Gröbner 基，可消元得到单变量多项式，再因式分解取得 $r$：

```python
p, q, r = ZZ["p,q,r"].gens()
I = ideal(
    p * (2*q + 1) - N1,
    q * (2*r + 1) - N2,
    r * (2*p + 1) - N3,
)
B = I.groebner_basis()
print(B[3].factor())
```

得到 $r$ 后直接回代：

```python
p = (N3 // r - 1) // 2
q = N2 // (2*r + 1)
```

三个模数的 Euler 函数分别为：

$$
\varphi(N_1)=2q(p-1),\quad
\varphi(N_2)=2r(q-1),\quad
\varphi(N_3)=2p(r-1).
$$

对 $e=65537$ 求三个逆元，解密 $c_i^{d_i}\bmod N_i$，再把三个 16 字节明文按顺序拼接，得到：

```text
grey{3_equations_3_unknowns_just_use_groebnerXD}
```

## 方法总结

每个 RSA 模数单独看都像正常的两素数乘积，但跨模数复用带代数关系的素数会把分解问题变成低维多项式求解。遇到多个相关模数时，除检查最大公因数外，还应把素数生成公式原样写成方程，尝试消元、结果式或 Gröbner 基，而不能把每个实例孤立分析。
