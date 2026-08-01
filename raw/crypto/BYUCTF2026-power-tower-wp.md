# Power Tower

## 题目简述

题目使用 25 个互不相同的 16 位素数构造多素数 RSA 模数 $n$，并把 25 个小整数组织成右结合幂塔。若幂塔为 $E=e_1^{e_2^{\cdot^{\cdot^{e_{25}}}}}$，服务端返回 $c=m^E\bmod n$，要求在一秒内恢复 $m$。关键提示是每个素因子均不超过 $2^{16}$，所以 $n$ 虽然很大，却能用预生成的小素数表快速试除。

## 解题过程

先筛出 $2^{16}$ 以内的素数并试除 $n$，即可得到完整分解。由于 $n$ 是平方自由的多素数模数，

$$
\varphi(n)=\prod_i(p_i-1).
$$

计算巨大幂塔时不能展开指数。利用欧拉降幂，求 $a^b\bmod M$ 时只需知道 $b\bmod\varphi(M)$；对指数本身继续递归，便需要依次得到

$$
\varphi(n),\ \varphi^2(n),\ \ldots,\ \varphi^{25}(n).
$$

每一层的当前模数都以质因数幂字典表示。对 $p^k$ 使用 $\varphi(p^k)=p^{k-1}(p-1)$，再把每个 $p-1$ 试除分解，就能快速生成下一层因子表。

设 $R=e_2^{\cdots^{e_{25}}}$。加密指数是 $e_1^R$，其模 $\varphi(n)$ 的逆元为

$$
d=(e_1^{-1})^R\bmod\varphi(n).
$$

官方生成器保证 $\gcd(e_1,\varphi(n))=1$。因此只需把幂塔首项替换为 `pow(e1, -1, phi_n)`，再用同一递归降幂函数计算 $d$：

```python
primes = generate_primes_up_to(2**16)
factors = factorize_with_known_primes(n, primes)
phis = compute_phis(factors, len(exps), primes)

inverse_tower = exps.copy()
inverse_tower[0] = pow(exps[0], -1, phis[0])
d = power_tower_mod_n(inverse_tower, phis[0], phis[1:])
m = pow(c, d, n)
```

把恢复的十进制 $m$ 在时限内提交后，服务返回：

```text
byuctf{eulers_phi_phunction_is_a_phun_phunction}
```

## 方法总结

- 核心技巧：先利用小素因子上界分解多素数模数，再以迭代欧拉函数递归计算模幂塔及其逆指数。
- 识别信号：题目同时出现“所有因子很小”“幂塔指数”和严格时限时，重点是预筛素数与模数链，而不是构造完整指数。
- 复用要点：始终保持每层 $\varphi^k(n)$ 的质因数分解；直接构造幂塔会立即失控。还应先确认幂塔首项与 $\varphi(n)$ 互素，逆元才存在。
