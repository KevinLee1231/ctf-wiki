# dlog24

## 题目简述

题目在商环

$$
R=(\mathbb Z/p^{24}\mathbb Z)[x]/(f(x))
$$

中公开 $c=x^m$，其中 $m$ 是 flag 的大整数表示。表面上这是模 $p^{24}$ 的高阶离散对数，但生成脚本同时给出了把同一结构提升到未分歧 $p$ 进域的直接方法。

## 解题过程

多项式 $f$ 是 $\mathbb F_{p^{24}}$ 生成元最小多项式的整数提升。用同一模多项式构造精度 24 的 $p$ 进扩张：

```sage
R = Zmod(p^24)[x].quo(f)
K.<a> = Qq(p^24, modulus=f, prec=24)
```

将题目给出的商环元素 $c=\sum c_i x^i$ 按系数嵌入 $K$：

```sage
lifted = sum(ZZ(coeff) * a^i for i, coeff in enumerate(c.list()))
```

在 $p$ 进域中，对接近 Teichmüller 单位的元素可以使用 $p$ 进对数；生成端正是用 `lifted.log()/a.log()` 取回指数。因此求解为：

```sage
m = ZZ(lifted.log() / a.log())
print(m.to_bytes(m.bit_length() // 8 + 1, "big"))
```

得到：

```text
grey{h3h3heheh3h3_1_luv_p0lynom1als_s0_much_sie_ist_me1ne_best1e!1!11!!11!!1!!!11!!11!}
```

## 方法总结

指数运算所在的环并不自动提供离散对数安全性。这里模 $p^{24}$ 的系数环和最小多项式明确暴露了一个 $p$ 进提升，乘法关系在嵌入后仍成立，而 $p$ 进对数把乘法变为加法：$\log(a^m)=m\log(a)$。看到素数幂模数、扩域最小多项式和精度相符时，应优先检查 Hensel／$p$ 进提升，而不是把它当作普通有限域 DLP。
