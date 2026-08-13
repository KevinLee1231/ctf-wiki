# GreyCTF2022 - Equation 3

## 题目简述

第三题继续使用两个 flag 分块和低次多项式关系，但 RSA 运算位于复合模数 $N$ 上。不能再直接使用素数域乘法群的阶；官方解法在模 $N$ 的多项式商环里计算幂并消元，最后对小明文块应用 Coppersmith。

## 解题过程

由源码确定变量、公开关系和指数后，在 $\mathbb Z/N\mathbb Z$ 上建立理想 $I=\langle f,g\rangle$。将涉及 $m_1,m_2$ 的幂在商环中约化，可以控制表达式次数；随后通过结式逐个消去 $m_1,m_2$，得到关于明文候选 $k$ 的一元模多项式。

```sage
R.<m1, m2> = PolynomialRing(Zmod(N))
I = R.ideal([f, g])
Q = R.quotient(I)

# 在 Q 中计算并约化目标幂，再提升回多项式进行 resultant 消元
...
root = univariate.monic().small_roots(X=bound, beta=1)[0]
```

分别恢复两半后转成字节并拼接。官方数据得到：

```text
grey{Hope_you_like_this_equation_series_:D_k9!LY$jWT&1*%wjPKCo2EnsXB3}
```

其中 `$`、`*`、`%` 都是 flag 的普通字符，不应被当成 Markdown 数学或格式标记。

## 方法总结

复合模数上的多项式运算要区分“在商环中降次”和“在整数或模环中找小根”两个阶段。消元会迅速放大次数和系数，实际解题应缓存中间多项式；恢复结果必须重新加密并检查全部关系，避免接受由结式引入的伪根。
