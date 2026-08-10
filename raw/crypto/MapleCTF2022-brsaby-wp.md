# brsaby

## 题目简述

题目给出标准 RSA 参数 $N=pq$、$e=65537$、密文 $c$，同时额外泄露

$$
h=p^4-q^3.
$$

单独看这个提示不像常见的 $p+q$ 或高位泄露，但它和 $N=pq$ 可以消去 $q$，把分解问题转化为一个具有整数根的单变量多项式问题。

## 解题过程

由 $q=N/p$ 代入提示式，有

$$
h=p^4-\left(\frac{N}{p}\right)^3.
$$

两边乘以 $p^3$ 后整理：

$$
p^7-hp^3-N^3=0.
$$

因此在整数环上构造

```python
P = PolynomialRing(Integers(), "x")
x = P.gen()
f = x**7 - hint*x**3 - N**3
p = f.roots()[0][0]
q = N // p
assert p * q == N
```

这里的目标不是数值近似求根，而是让 Sage 在整数环中找精确根；真实素数 $p$ 必然是该多项式的整数根。得到 $p,q$ 后按普通 RSA 解密：

```python
phi = (p - 1) * (q - 1)
d = inverse_mod(e, phi)
m = power_mod(c, d, N)
```

把 $m$ 转为字节即可得到：

```text
maple{s0lving_th3m_p3rf3ct_r000ts_1s_fun}
```

## 方法总结

RSA 的额外提示首先应与 $N=pq$ 联立，而不是孤立地尝试开根或格攻击。本题的关键是清除分母后识别“已知系数、多项式含精确整数根”的结构。若使用浮点根求解，数百位整数会造成严重精度问题；应直接在 Sage 的整数多项式环中求根并用 $p\mid N$ 复核。
