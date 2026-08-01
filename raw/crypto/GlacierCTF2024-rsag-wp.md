# GlacierCTF 2024 Rivest–Shamir–Adleman–Germain

## 题目简述

题目给出 textbook RSA 的模数 $N$ 和密文 $CT$。模数并非由四个独立随机素数构成，而是从 Sophie Germain 形式连续派生：

$$
q=2p+1,\qquad r=2q+1,\qquad s=2r+1.
$$

四个因子只包含一个未知量 $p$，因此可以把模数写成关于 $p$ 的四次多项式并直接求整数根，随后按普通 RSA 解密。

## 解题过程

### 1. 将全部素数写成 $p$ 的函数

递推关系展开为：

$$
q=2p+1,\qquad r=4p+3,\qquad s=8p+7.
$$

代入 $N=pqrs$：

$$
N=64p^4+136p^3+94p^2+21p.
$$

所以应求解：

$$
64x^4+136x^3+94x^2+21x-N=0.
$$

官方 Sage solver 使用 `solve()` 并从实根中选择通过 `isPrime()` 的正整数；由于多项式在正区间严格递增，也可以对 $p$ 做整数二分或从四次根附近校正，最后用 $p(2p+1)(4p+3)(8p+7)=N$ 精确验证。

### 2. 重建私钥并解密

得到 $p$ 后依次计算 $q,r,s$，再求：

$$
\varphi(N)=(p-1)(q-1)(r-1)(s-1),
$$

$$
d\equiv65537^{-1}\pmod{\varphi(N)},\qquad
m\equiv CT^d\pmod N.
$$

核心代码为：

```python
q, r, s = 2*p + 1, 4*p + 3, 8*p + 7
assert p*q*r*s == N
phi = (p-1) * (q-1) * (r-1) * (s-1)
d = pow(65537, -1, phi)
m = pow(CT, d, N)
flag = m.to_bytes((m.bit_length() + 7) // 8, "big")
```

用仓库输出复算得到：

```text
gctf{54dly_50ph13_63rm41n_pr1m35_wh3r3_n07_u53d_53curly}
```

## 方法总结

RSA 要求模数因子在攻击者视角下彼此不可预测；“每个因子都是素数”并不等于安全。这里四个 512 位素数只有约 512 位的共同自由度，公开模数就给出一个可精确求根的低次方程。遇到结构化素数时，应先代数消元并验证候选因子，而不是直接尝试通用大整数分解。
