# GlacierCTF 2025 Noisy Neighbour

## 题目简述

题目生成两个 1024 位素数 $p,q$，令 $N=pq$、$\varphi=(p-1)(q-1)$，但公钥指数不是常见的 $65537$，而是

$$
e=k\varphi+r,
$$

其中 $k$ 只有约 128 位，$r$ 是约 16 位的小素数。服务先用 RSA 加密一个 128 位 ticket，再将 ticket 作为 AES-CTR 密钥加密 flag。

这意味着 $e$ 是 $\varphi(N)$ 的一个带小误差倍数。只要从这个近似倍数恢复 $p,q$，后续就是标准 RSA 与 AES 解密。

## 解题过程

### 1. 从 $e/N$ 估计倍数 $k$

由于 $\varphi=N+1-(p+q)$ 且非常接近 $N$，有

$$
\frac eN=\frac{k\varphi+r}{N}\approx k.
$$

因此先取 `k0 = e // N`。取整误差只会让真实 $k$ 比 `k0` 大很小的常数，官方 Sage solver 依次尝试

```python
for delta in range(7):
    k = e // N + delta
```

即可覆盖实例。

### 2. 近似恢复 $p$，再用 Coppersmith 修正

由 $e=k\varphi+r$ 可得

$$
\frac ek=\varphi+\frac rk,
$$

而 $r/k$ 很小，于是可以估计

$$
s=p+q=N+1-\varphi\approx N+1-\left\lfloor\frac ek\right\rfloor.
$$

若 $s$ 精确，较大的素因子就是二次方程 $X^2-sX+N=0$ 的根：

$$
p=\frac{s+\sqrt{s^2-4N}}2.
$$

用近似的 $s$ 算出 `p_approx` 后，真实值可写成 $p=p_0+x$，其中误差 $x$ 足够小。构造一次多项式

$$
f(x)=x+p_0
$$

并在模 $N$ 下寻找其小根。因为真实 $p$ 是 $N$ 的一个约 $N^{1/2}$ 大因子，Sage 中可直接使用：

```sage
PR.<x> = PolynomialRing(Zmod(N))
roots = (x + p_approx).small_roots(beta=0.5, epsilon=0.01)
```

对每个小根检查 `gcd(N, p_approx + root)`；得到非平凡因子即完成分解。这个思路对应题目引用的 [ePrint 2025/2079](https://eprint.iacr.org/2025/2079)：已知一个接近 $\varphi(N)$ 倍数的整数时，可以把因子恢复转化为小根问题。正文中的近似式已经给出了本题所需的具体化过程，无需依赖论文才能复现。

### 3. 解开两层密文

恢复 $p,q$ 后重新计算 $\varphi$，并求

$$
d=e^{-1}\bmod\varphi.
$$

对 RSA 密文计算 `pow(c, d, N)`，按 16 字节恢复 ticket；再以 ticket 为 AES 密钥、题目给出的 nonce 创建 CTR cipher，解密第二段密文即可。源码实例得到：

```text
gctf{c0pp3rsm17h_ste4ling_y0ur_sm4ll_r00ts_2025-2079}
```

## 方法总结

异常大的 RSA 指数本身不是漏洞，决定性问题是它被构造成 $k\varphi+r$，且 $k$、$r$ 都远小于模数。先用 $N\approx\varphi$ 确定 $k$ 的极小搜索范围，再从 $p+q$ 的近似值获得一个接近真实素因子的整数，最后交给 Coppersmith 修正小误差。分解成功后必须检查因子整除关系和 RSA 逆元，再进行 AES 解密，避免把数值近似误判为有效分解。
