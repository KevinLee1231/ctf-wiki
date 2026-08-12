# DownUnderCTF 2020 - 1337crypt

## 题目简述

题目用一个 Goldwasser–Micali 风格方案逐位加密 flag。公钥模数为两个 1337 位素数的乘积 $n=pq$，已知值 $x$ 在模 $p$、模 $q$ 下都是二次非剩余。只有分解 $n$ 后才能用二次剩余性判断明文位；附件额外给出：

$$
h=\left\lfloor D(\sqrt p+\sqrt q)\right\rfloor,
\qquad D=(1\cdot3\cdot3\cdot7)^{1+3+3+7}.
$$

## 解题过程

### 从平方根和得到素因子近似值

忽略取整产生的小误差，$D\sqrt p$ 与 $D\sqrt q$ 近似是多项式

$$
X^2-hX+D^2\sqrt n
$$

的两个根。取其中一个根 $X_0$，计算

$$
p'\approx\left(\frac{X_0}{D}\right)^2,
$$

可以得到约 750 个高位正确的 $p$，剩余误差满足 $p=p'+\delta$ 且 $|\delta|<2^{590}$。

### 用 Coppersmith 补回低位

在 $\mathbb Z_n[x]$ 中构造首一多项式：

$$
f(x)=x+p'.
$$

真实误差 $\delta$ 满足 $f(\delta)=p$，所以它是模 $n$ 的未知大因子 $p$ 上的小根。以 $X=2^{590}$、$\beta=0.4$ 调用 Coppersmith 小根算法即可恢复 $\delta$，进而得到精确素因子。

```sage
from Crypto.Util.number import long_to_bytes

# 载入题目给出的 hint、D、n、c
exec(open("output.txt", encoding="utf-8").read())

R.<x> = PolynomialRing(SR)
approx_poly = x^2 - hint * x + D^2 * sqrt(n)
approx_root = approx_poly.roots()[0][0]
p_approx = ZZ((approx_root / D)^2)

P.<y> = PolynomialRing(Zmod(n), implementation="NTL")
roots = (y + p_approx).small_roots(
    X=2^590,
    beta=0.4,
    epsilon=1/32,
)
p = ZZ(p_approx + roots[0])
assert n % p == 0 and is_prime(p)
```

### 按二次剩余性还原每一位

每个密文位的形式是：

$$
c_i=x^{1337+b_i}r^{2674}\bmod n.
$$

$r^{2674}$ 必为二次剩余，而 1337 是奇数。于是 $b_i=0$ 时 $x^{1337}$ 为非剩余，$b_i=1$ 时 $x^{1338}$ 为剩余。对恢复出的 $p$ 计算 Kronecker/Legendre 符号即可逐位解密：

```sage
bits = ''.join('0' if kronecker(value, p) == -1 else '1' for value in c)
flag = long_to_bytes(int(bits, 2))
print(flag.decode())
```

结果为：

```text
DUCTF{wh0_N33ds_pr3cIsi0n_wh3n_y0u_h4v3_c0pp3rsmiths_M3thod}
```

## 方法总结

本题先用平方根和泄漏得到素因子的高精度近似，再把未知低位写成模未知因子的“小根”。看到 $n=pq$、近似素因子或高位泄漏时，应估计误差位数并检查 Coppersmith 条件；分解完成后，还要按实现中的奇偶指数重新判断二次剩余与明文位的映射，不能照搬标准 GM 方案。
