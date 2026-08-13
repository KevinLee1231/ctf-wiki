# GreyCTF2022 - Equation 2

## 题目简述

第二题先在素数域中对两个 flag 分块执行 RSA 指数 $e=65537$，再只公开关于加密后变量的两条低次多项式关系。直接枚举不可行，官方解法把模逆幂、商环消元和 Coppersmith 小根串成一条恢复链。

## 解题过程

设明文块为 $x_1,x_2$，密文变量为 $m_i=x_i^e\bmod p$。先在

$$\mathbb F_p[m_1,m_2]/\langle f,g\rangle$$

中计算逆指数 $d=e^{-1}\bmod(p-1)$，把 $m_1^d,m_2^d$ 约化为低次基上的线性组合。随后引入新变量 $k$ 表示候选明文块，对这些表达式连续取结式，得到只含 $k$ 的一元四次多项式。

官方脚本已保存两条消元后的多项式；将它们首一化后调用小根算法：

```sage
F.<k> = GF(p)[]
poly = poly / poly.leading_coefficient()
root = poly.small_roots(epsilon=0.03)[0]
print(long_to_bytes(ZZ(root)))
```

两次恢复分别输出：

```text
grey{Equations_are_beautify_
arent_they_Gb9kkmVPHDUNpJCU}
```

拼接得到 `grey{Equations_are_beautify_arent_they_Gb9kkmVPHDUNpJCU}`。

## 方法总结

该题展示了“先在商环中表示逆运算，再在整数域中消元并利用明文小于模数的界”这一组合。Coppersmith 不是替代建模的黑盒：多项式必须在正确模数上首一化，根界必须满足假设，得到的块也要重新代入加密和两条公开方程验证。
