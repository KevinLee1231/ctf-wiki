# meow-log-meow-log-meow-e

## 题目简述

题目给出同一 RSA 模数下的两个密文：

$$
c_1=m^e\bmod n,\qquad c_2=E(m)^e\bmod n,
$$

其中 $e=65537$，而 $E(x)=\sum_{i=0}^{t-1}x^i/i!$ 是在 $\mathbb Z_n$ 上截断的“指数函数”。这是高次数版本的 Franklin–Reiter 相关消息攻击；普通多项式欧几里得算法会慢到不可用，因此官方解法使用 half-GCD。

## 解题过程

在 $R=(\mathbb Z/n\mathbb Z)[x]$ 中构造两个都以真实明文 $m$ 为根的多项式：

$$
f_0(x)=x^e-c_1,
$$

$$
f_1(x)=E(x)^e-c_2.
$$

二者的最大公因式应包含 $x-m$。直接逐次取余处理次数 $65537$ 的稠密多项式非常慢；half-GCD 每次只看多项式的高半部分，递归求一个 $2\times2$ 变换矩阵，把两项的次数近似减半，再回到欧几里得步骤：

```sage
F = Zmod(n)
R.<x> = PolynomialRing(F)
f0 = x^e - c1
f1 = e_power(x, F, po)^e - c2
g = poly_gcd(f0, f1, n).monic()
```

当结果为一次首一多项式 $g(x)=x+g_0$ 时：

```sage
m = int(-g[0] % n)
print(long_to_bytes(m))
```

输出：

```text
grey{me0w_m3ow_s0lUtioN_t0o_Sl0w_4_m3-oW!!!}
```

## 方法总结

RSA 相关消息攻击并不限于线性关系；只要第二个消息是已知低于模数语义的多项式函数，就能在 $\mathbb Z_n[x]$ 中构造公共根。此题的难点是算法工程：$e=65537$ 让普通 GCD 超过一天，而 divide-and-conquer half-GCD 把次数下降过程加速到可用范围。实现时还要注意 $\mathbb Z_n$ 不是域，某些首项系数可能不可逆；但出现这种情况往往反而能通过与 $n$ 求 gcd 分解模数。
