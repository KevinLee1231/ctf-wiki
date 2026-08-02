# copper-master

## 题目简述

服务端使用 $e=3$ 的 RSA，并把 200 位随机消息 $m$ 填充为

$$
M=31415\cdot2^{1385}+m^2\cdot2^{500}+m,
$$

随后给出 $c=M^3\bmod n$。填充完全由小消息的低次多项式决定，适合用 Coppersmith 小根攻击。

## 解题过程

模数约 1536 位，而未知量满足 $m<2^{200}$。在 $\mathbb Z_n[x]$ 上构造：

```sage
R.<x> = PolynomialRing(Integers(n))
f = (2^1385 * 31415 + 2^500 * x^2 + x)^3 - c
f = f.monic()
roots = f.small_roots(X=2^200, epsilon=1/(f.degree()*4))
```

真实消息满足 $f(m)\equiv0\pmod n$，且相对模数足够小。`small_roots` 建格后通过 LLL 找到对应的小整数根，无需分解 $n$。把根作为十进制整数提交给同一连接，服务端与内部随机消息比较成功并返回：

```text
tjctf{I_gu3ss_CusT0m_p4dd1ngs_b4d_2}
```

这里的 `monic()` 很重要：Sage 的小根实现需要可逆的首项系数；边界 $X=2^{200}$ 则直接来自 `os.urandom(25)`。

## 方法总结

- RSA 填充必须引入不可预测随机性；确定性的低次多项式填充会把明文变成模方程的小根。
- Coppersmith 的输入不是“尝试分解 RSA”，而是明确写出 $f(m)\equiv0\pmod n$ 并给出可信的根上界。
- 源码中消息位数、指数和填充次数共同决定多项式次数与攻击可行性。
