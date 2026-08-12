# babyRSA

## 题目简述

题目使用标准 RSA 指数 $e=65537$，但模数不是两个素数的乘积，而是：

$$
N=p^2q,
$$

其中 $p,q$ 均为 $1024$ 位素数。公开输出给出 $N,e,c$，以及

$$
p_{\mathrm{msb}}=(p\mathbin{\texttt{>>}}320)\mathbin{\texttt{<<}}320.
$$

也就是说，$p$ 的高 $704$ 位已知，未知低位满足 $p=p_{\mathrm{msb}}+x$ 且 $0\le x<2^{320}$。因 $p$ 是 $N$ 的约 $N^{1/3}$ 因子，泄露位数足以对线性多项式做 Coppersmith small-roots；恢复 $p$ 后即可得到私钥。决定性障碍是部分已知 RSA 因子的格攻击，故归入 `crypto`。

## 解题过程

### 选择正确的 Coppersmith 参数

在环 $\mathbb{Z}_N[x]$ 中构造：

$$
f(x)=p_{\mathrm{msb}}+x.
$$

真实低位 $x_0=p-p_{\mathrm{msb}}$ 使 $f(x_0)=p$，所以 $f(x_0)\equiv0\pmod p$。注意这里不应把根误认为模 $p^2$：$f(x_0)$ 等于 $p$，只保证被 $p$ 整除。由于 $p\approx N^{1/3}$，给 Sage 的 factor-size 参数取 $\beta\approx1/3$；已知界为 $X=2^{320}$。

```sage
PR.<x> = PolynomialRing(Zmod(N))
f = x + p_msb
roots = f.small_roots(X=2^320, beta=0.33, epsilon=0.01)
assert roots

x0 = int(roots[0])
p = p_msb + x0
assert N % (p * p) == 0
q = N // (p * p)
```

`small_roots` 的格约化在这里找的是低于 $2^{320}$ 的小根，而不是分解整个 $3072$ 位模数。最后的整除断言把候选根和实际模数结构重新绑定，排除近似格结果。

### 用重复素因子计算私钥

对于互素的 $p^2$ 和 $q$，Euler 函数为：

$$
\varphi(N)=\varphi(p^2)\varphi(q)=p(p-1)(q-1).
$$

因此可按通常 RSA 解密：

```python
from Crypto.Util.number import long_to_bytes

phi = p * (p - 1) * (q - 1)
d = inverse_mod(e, phi)
m = pow(c, d, N)
flag = long_to_bytes(int(m)).decode()
print(flag)
```

官方 Sage solver 恢复并输出：

```text
grey{th1s_15_pr0b4bly_t00_34sy_n0w4d4y5_1n34v80n23}
```

### 验证恢复结果

完整性检查应至少包括 `N % (p*p) == 0`、`is_prime(p)`、`is_prime(q)`，以及重新计算 $m^e\bmod N$ 是否等于公开 $c$。这些检查能区分“small_roots 返回了一个数值候选”和“确实恢复了 RSA 因子”。

## 方法总结

- 核心技巧：对 $N=p^2q$，较小的 $p$ 约为 $N^{1/3}$；已知其高 $704$ 位时，未知 $320$ 位可作为 $f(x)=p_{\mathrm{msb}}+x$ 的 Coppersmith 小根恢复。
- 识别信号：RSA 模数含重复素因子、公开了某个因子的 MSB、且未知尾部明显短于因子位长时，应比较 $X$ 与 $N^{\beta^2}$ 的量级并尝试 univariate `small_roots`。
- 复用要点：`beta` 对应“模数中保证整除 $f(x_0)$ 的因子大小”，不是随意的调参。恢复因子后必须使用 $\varphi(p^2q)=p(p-1)(q-1)$，不能错误套用 $(p-1)(q-1)$。
