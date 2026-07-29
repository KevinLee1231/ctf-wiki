# マスタースパーク (Master Spark)

## 题目简述

服务生成一个 $256$ 位素数 `secret`。玩家可多次提交不超过 $96$ 位的素数 $p$，服务在

$$
E_A:\ y^2=x^3+Ax^2+x
$$

上执行两方 CSIDH 风格的交换，并返回共享曲线上的两个点 $P,Q$。由于 isogeny 映射保持标量乘法，返回点满足：

$$
Q=\text{secret}\cdot P.
$$

提交的 $p$ 需要满足特殊分解条件：$(p+1)/4$ 中恰有一个素因子的指数大于 $1$，该底数不超过 $32$ 位；其余素因子均不超过 $16$ 位，且所有查询之间不能重复使用素因子。

## 解题过程

### 主动选择平滑子群

攻击者可以自行决定曲线群阶相关的 $p+1$。构造：

$$
p=4r^e\prod_i \ell_i-1,
$$

其中 $r^e$ 接近 $2^{32}$，$\ell_i$ 是若干互不重复的小素数，并不断尝试直到 $p$ 为素数。这样每次查询都能得到一个新的、易做离散对数的素数幂模数 $r^e$。

官方脚本构造约十个这样的 $p$，并确保各查询使用的因子互不重叠，以通过服务端的 `choice` 检查。

### 在目标素数幂子群中求离散对数

由返回的 $P=(x_P,y_P)$ 可恢复 Montgomery 曲线参数：

$$
A=\frac{y_P^2-x_P^3-x_P}{x_P^2}\pmod p.
$$

随后把 $P,Q$ 放回曲线。对群阶 `order = p + 1` 和目标因子 $r^e$，先投影到该子群，再逐位执行 Pohlig–Hellman：

```python
def pohlig_hellman_prime_power(Q, P, order, subgroup_order):
    P = (order // subgroup_order) * P
    Q = (order // subgroup_order) * Q
    base, exp = factor(subgroup_order)[0]
    P0 = (order // base) * P
    z = 0

    for i in range(1, exp + 1):
        Q0 = (order // (base**i)) * (Q - z * P)
        for k in range(base):
            if Q0 == k * P0:
                z += k * base**(i - 1)
                break
    return int(z)
```

每次查询可恢复：

$$
\text{secret}\equiv \pm d_i\pmod{r_i^{e_i}}.
$$

符号不确定性来自只依据点坐标和曲线表示恢复关系时，$Q$ 与 $-Q$ 都可能对应同一个 $x$ 方向。

### CRT 合并

持续收集互素的 $r_i^{e_i}$，直到它们的乘积覆盖至少 $256$ 位。枚举每项的正负两种余数，用 CRT 合并：

```python
for residues in product(*[(d, modulus - d)
                          for d, modulus in samples]):
    candidate = CRT(residues, moduli)
    if candidate.nbits() <= 256 and isPrime(candidate):
        secret = int(candidate)
        break
```

题目保证真正的 `secret` 是 $256$ 位素数，因此位数和素性足以从符号组合中筛出正确候选。把该值提交给服务即可获得 flag。

## 方法总结

- 核心技巧：主动选择带平滑素数幂因子的曲线群阶，把隐藏标量分别投影到多个易解子群，再用 Pohlig–Hellman 与 CRT 恢复。
- 识别信号：服务允许攻击者选择有限域素数或群阶，而输出点之间保留同一个秘密标量关系。
- 复用要点：限制单次群阶大小并不能阻止多次 CRT 泄漏；协议必须固定安全参数，或至少阻止攻击者控制能够承载秘密标量的子群结构。
