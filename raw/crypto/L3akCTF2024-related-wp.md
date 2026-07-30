# L3akCTF 2024 Related Writeup

## 题目简述

题目在同一个 RSA 模数 $n$ 和指数 $e=0x101$ 下，将同一 flag 分别做两次已知仿射变换：

$$
m_1=a_1m+b_1,\qquad m_2=a_2m+b_2,
$$

并公开 $a_1,b_1,a_2,b_2$ 以及：

$$
c_1\equiv m_1^e\pmod n,\qquad
c_2\equiv m_2^e\pmod n.
$$

随机系数没有提供安全填充；两个明文仍是关于同一未知量 $m$ 的已知低次关系，可用 Franklin–Reiter related-message attack 恢复。

## 解题过程

在多项式环 $(\mathbb Z/n\mathbb Z)[X]$ 中构造：

$$
f_1(X)=(a_1X+b_1)^e-c_1,
$$

$$
f_2(X)=(a_2X+b_2)^e-c_2.
$$

真实明文 $m$ 同时满足 $f_1(m)=f_2(m)=0$。在通常条件下，两式的最大公因式就是与 $X-m$ 等价的一次多项式。

Sage 在合数模数多项式环上可能不能直接调用标准 GCD，因此官方脚本使用首一化的欧几里得算法：

```sage
def poly_gcd(a, b):
    while b:
        a, b = b, a % b
    return a.monic()

P.<X> = PolynomialRing(Zmod(n))
f1 = (a1 * X + b1)^e - c1
f2 = (a2 * X + b2)^e - c2
g = poly_gcd(f1, f2)
m = int(-g[0])
```

将 $m$ 转回字节后得到：

```text
L3AK{r3l4teD_m3s54GeS_Ar3_1nS3cuR3_1n_RsA}
```

## 方法总结

- 核心技巧：把两个已知仿射相关的 RSA 明文写成同模多项式，通过多项式 GCD 提取共同根。
- 识别信号：同一模数和指数下重复加密 $a_im+b_i$，即使每次 $a_i,b_i$ 随机，只要这些系数公开，就不是安全的随机填充。
- 复用要点：合数模数上的多项式环不是域，库函数可能在求逆或首一化时失败；可使用定制欧几里得过程，并检查最终 GCD 是否确为一次式。
