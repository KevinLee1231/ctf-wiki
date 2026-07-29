# HEuristic

## 题目简述

服务使用 Microsoft SEAL 的 CKKS 参数，每次连接生成新密钥，并随机选择：

$1\le\delta<q$

其中 $q$ 是五个 48 位系数模数的乘积，约为 240 位。玩家只有三次菜单操作，必须依次完成一次加密、一次解密和一次提交。

加密接口接收 4096 个整数系数 $m_i$，却在构造明文时额外乘上秘密 $\delta$。解密接口返回前 96 个系数，并为每项增加绝对值小于

$B=5\cdot2^{185}$

的随机噪声。由于 $B$ 约为 188 位，每个 240 位模 $q$ 观测仍泄漏约 52 位精度。构造 95 个共享同一基线噪声的近似模方程后，可以用区间细化恢复精确的 $\delta$。

这不是 SEAL 或 CKKS 本身的密码分析漏洞，而是应用把过高精度的解密结果作为 oracle 返回。

## 解题过程

### 1. 写出实际泄漏方程

服务手工构造 RNS 明文：

```cpp
plain[i + j * coeff_count] =
    multiply_mod(coeff_rns[i][j], delta_rns[j], moduli[j]);
```

随后做 NTT 并用公钥加密。把得到的合法密文原样送回解密接口后，逆 NTT 与 RNS 合成会恢复：

$m_i\delta\pmod q$

解密接口再加或减独立随机噪声：

```cpp
cpp_int noise_bound = cpp_int(5) << 185;
cpp_int noise = random_below(noise_bound, noise_rng);

value = (coeffs[i] ± noise) % q;
```

把 SEAL 自身远小于该上界的解密误差也并入噪声，可写成：

$\ell_i\equiv m_i\delta+\varepsilon_i\pmod q$

其中：

$|\varepsilon_i|<B$

服务只隐藏下标 96 之后的系数，所以一次密文仍给出 96 个方程。

### 2. 绕过“明文不能接近零”的限制

加密接口拒绝中心代表绝对值小于 $q/8$ 的系数：

```cpp
cpp_int reduced = coeff % q;
cpp_int abs_coeff = reduced > q / 2 ? q - reduced : reduced;
if (abs_coeff < q / 8) {
    throw std::invalid_argument("bad plaintext");
}
```

因此不能直接提交 $1,2,4,\ldots$。选择允许的公共基线：

$b=\lfloor q/2\rfloor$

并构造：

$m_0=b$

$m_e=b+2^e$

参考解法使用：

```python
exponents = [e for e in range(0, 192, 2) if e != 74]
```

该集合恰有 95 项。所有 $b+2^e$ 仍远离模 $q$ 的零点，可以通过检查。4096 槽布局为：

```python
coeffs = [q // 2]
coeffs.extend(q // 2 + (1 << e) for e in exponents)
coeffs.extend([q // 2] * (4096 - len(coeffs)))
```

### 3. 相减消去大基线

第一项泄漏为：

$\ell_0\equiv b\delta+\varepsilon_0\pmod q$

其余项为：

$\ell_e\equiv(b+2^e)\delta+\varepsilon_e\pmod q$

两者相减：

$d_e=(\ell_e-\ell_0)\bmod q$

$d_e\equiv2^e\delta+(\varepsilon_e-\varepsilon_0)\pmod q$

并且：

$|\varepsilon_e-\varepsilon_0|<2B$

这一步既消除了无法直接使用的小明文系数，也把 95 个观测转成同一个未知数 $\delta$ 的近似模乘关系。

### 4. 把一个模约束反投影为整数区间

对固定 $e$，真实余数必须落在圆环区间：

$[d_e-2B,d_e+2B]\pmod q$

若当前 $\delta$ 候选区间为 $[L,H]$，令 $a=2^e$，则：

$aL\le a\delta\le aH$

同时存在整数 $k$ 与允许余数 $r$ 使：

$a\delta=kq+r$

对每个可能的环绕次数 $k$，新的整数区间为：

$$
\left\lceil\frac{kq+r_{\min}}a\right\rceil
\le\delta\le
\left\lfloor\frac{kq+r_{\max}}a\right\rfloor
$$

与 $[L,H]$ 取交即可。若噪声区间跨越 0，还要先拆成：

```text
[0, high] 和 [q + low, q - 1]
```

核心实现如下：

```python
def split_residue_interval(center, radius, q):
    center %= q
    lo = center - radius
    hi = center + radius
    if lo < 0:
        return [(0, hi), (q + lo, q - 1)]
    if hi >= q:
        return [(lo, q - 1), (0, hi - q)]
    return [(lo, hi)]


def intersect_constraint(intervals, exponent, observed, radius, q):
    multiplier = 1 << exponent
    out = []

    for lo_delta, hi_delta in intervals:
        lo_product = multiplier * lo_delta
        hi_product = multiplier * hi_delta

        for lo_residue, hi_residue in split_residue_interval(
            observed, radius, q
        ):
            k_min = (lo_product - hi_residue + q - 1) // q
            k_max = (hi_product - lo_residue) // q

            for k in range(k_min, k_max + 1):
                lo_num = k * q + lo_residue
                hi_num = k * q + hi_residue
                new_lo = max(
                    lo_delta,
                    (lo_num + multiplier - 1) // multiplier,
                )
                new_hi = min(
                    hi_delta,
                    hi_num // multiplier,
                )
                if new_lo <= new_hi:
                    out.append((new_lo, new_hi))

    out.sort()
    merged = []
    for lo, hi in out:
        if merged and lo <= merged[-1][1] + 1:
            merged[-1] = (
                merged[-1][0],
                max(merged[-1][1], hi),
            )
        else:
            merged.append((lo, hi))
    return merged
```

从 `[(1, q - 1)]` 开始依次应用 95 个约束，候选通常会缩小到几个相邻整数，而不需要高维 LLL。

### 5. 利用共享的基线噪声筛掉相邻假解

区间细化只使用了保守界 $2B$，可能留下两三个候选。95 个差分并非拥有完全独立的差分噪声：每一个都包含相同的 $-\varepsilon_0$。

对候选 $\hat\delta$ 计算中心化残差：

$$
r_e=\operatorname{centered}
\left(d_e-2^e\hat\delta\bmod q\right)
$$

若候选正确，则存在同一个 $\varepsilon_0$，使所有：

$r_e=\varepsilon_e-\varepsilon_0$

都能由 $|\varepsilon_e|<B$ 解释，即：

$-B\le r_e+\varepsilon_0\le B$

因此对每个 $e$ 都得到：

$-B-r_e\le\varepsilon_0\le B-r_e$

将这些区间与初始 `[-B, B]` 全部求交。交集为空的候选必错；交集非空的候选再以残差平方和排序：

```python
def centered(value, q):
    value %= q
    return value - q if value > q // 2 else value


def solve_delta(q, leaks, exponents, noise_bound):
    baseline = leaks[0]
    diffs = [(value - baseline) % q for value in leaks[1:]]

    intervals = [(1, q - 1)]
    for exponent, observed in zip(exponents, diffs):
        intervals = intersect_constraint(
            intervals,
            exponent,
            observed,
            2 * noise_bound,
            q,
        )
        if not intervals:
            raise RuntimeError("all candidates were eliminated")

    candidates = []
    for lo, hi in intervals:
        for delta in range(lo, hi + 1):
            e0_lo = -noise_bound
            e0_hi = noise_bound
            residuals = []

            for exponent, observed in zip(exponents, diffs):
                residual = centered(
                    observed - ((1 << exponent) * delta % q),
                    q,
                )
                residuals.append(residual)
                e0_lo = max(e0_lo, -noise_bound - residual)
                e0_hi = min(e0_hi, noise_bound - residual)
                if e0_lo > e0_hi:
                    break

            if e0_lo <= e0_hi:
                midpoint = (e0_lo + e0_hi) // 2
                score = sum(
                    (residual + midpoint) ** 2
                    for residual in residuals
                )
                candidates.append((score, delta))

    if not candidates:
        raise RuntimeError("no common-noise candidate")
    candidates.sort()
    return candidates[0][1]
```

### 6. 完成三次菜单操作

交互顺序必须严格控制，因为服务总共只循环三次：

1. 读取开头输出的 `q`；
2. 选择 `1`，发送 4096 个上述系数；
3. 读取二进制 ciphertext 的精确长度和原始字节；
4. 选择 `2`，把相同 ciphertext 原样送回；
5. 解析输出的前 96 个整数；
6. 调用 `solve_delta()`；
7. 选择 `3` 并提交恢复值。

注意密文是二进制数据，不能使用按行读取替代定长读取。发送解密请求的格式为：

```python
sock.sendall(
    f"2\n{ciphertext_length}\n".encode()
    + ciphertext
    + b"\n"
)
```

本题仓库未附官方 solver；公开复现代码可见
[Abdelkad3r/R3CTF-2026 的 HEuristic solver](https://github.com/Abdelkad3r/R3CTF-2026/blob/master/crypto/HEuristic/solve.py)。
其外部内容的关键算法、边界和交互细节已经完整归纳在本文中，链接主要用于取得现成的 socket 封装。

## 方法总结

本题的恢复链为：

1. 加密接口实际构造 $m_i\delta$，解密接口泄漏其带界噪声版本；
2. 使用 $q/2$ 绕过“小明文禁止”检查；
3. 以 $q/2+2^e$ 构造 95 个槽，相减后得到 $2^e\delta$ 的近似模观测；
4. 把每个模圆环区间反投影到 $\delta$ 的整数区间并反复求交；
5. 利用所有差分共享同一个基线噪声 $\varepsilon_0$，筛掉相邻假解；
6. 在第三次菜单机会提交精确 $\delta$。

噪声只有在相对模数足够大时才能隐藏信息。这里 $q$ 约 240 位，噪声约 188 位，而服务一次返回 96 个相关样本；“每项都加了巨大随机数”并不等于安全，维度和共享结构会把剩余的约 52 位/样本迅速累积成完整秘密。
