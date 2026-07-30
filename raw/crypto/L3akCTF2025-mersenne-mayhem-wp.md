# L3akCTF 2025 Mersenne Mayhem Writeup

## 题目简述

题目取 Mersenne 素数

$$
p=2^n-1,\qquad n=11213,
$$

生成两个汉明重量均为 $w=10$ 的稀疏整数 $f,g$。它们的位长分别约为 $0.31n$ 和 $0.69n$，公钥为

$$
h=f g^{-1}\pmod p.
$$

加密 flag 时先计算

$$
\text{secret}=fg\bmod p,
$$

再令

$$
K=\operatorname{SHA3\_256}(\operatorname{I2OSP}(\text{secret}))
$$

并使用 AES-CBC。目标是只从 $(p,h)$ 恢复 $(f,g)$。

题目来自 AJPS 密码体制中的 Mersenne Low Hamming Ratio Search Problem，但 CTF 版本删去了真实方案的逐比特加密或 KEM、纠错编码等部分，只保留最直接的密钥恢复问题。

## 解题过程

### 从 AJPS 到本题的简化

AJPS 的基本公钥同样是

$$
h=f/g\pmod p,
$$

其中 $f,g$ 是低汉明重量整数。真实逐比特方案还会选取稀疏 $a,b$，以

$$
c=(-1)^m(ah+b)
$$

编码比特，并通过 $cg$ 的汉明重量判断消息；KEM 版本则发布类似 $(r,t=fr+g)$ 的量并配合纠错码。

本题没有这些防护层，差异可以概括为：

| 项目 | 真实 AJPS | 本题 |
| --- | --- | --- |
| 秘密重量 | 通常随 $\sqrt n$ 增长 | 固定 $w=10$ |
| 加密层 | 逐比特或 KEM，并带纠错编码 | 直接用 $fg$ 派生 AES 密钥 |
| 公开材料 | $h$ 或 KEM 的 $(r,t)$ | 只有 $h=f/g$ |
| 安全问题 | MLHRSP 与 MLHCSP | 直接求解 MLHRSP |
| 攻击目标 | 区分或消息恢复 | 恢复 $f,g$ 后解 AES |

因此只要恢复一次 $(f,g)$，整个加密链就结束了。

### 建立二元模小根

由公钥关系得到

$$
f-hg\equiv0\pmod p.
$$

定义齐次二元多项式

$$
F(x,y)=x-hy.
$$

真实根为 $(f,g)$，并有估计界

$$
|f|<X\approx p^{0.31},\qquad |g|<Y\approx p^{0.69}.
$$

官方 PDF 在概念说明中写成 $\xi_1+\xi_2<1$，但给出的十进制参数实际满足 $0.31+0.69=1$。源码的整数位长才是严格边界：

$$
\operatorname{bitlength}(f)\le\lfloor0.31n\rfloor=3476,
$$

$$
\operatorname{bitlength}(g)\le\lfloor0.69n\rfloor=7736,
$$

两者之和为 $11212<n$，故真实值仍满足 $fg<p$。求解时应以源码生成规则和最终同余校验为准。

### 构造格并约化

对参数 $m=t=s$，官方脚本构造 $m+1$ 个移位多项式

$$
G_k(x,y)=y^{m-k}F(x,y)^k p^{\max(t-k,0)},
\qquad k=0,\ldots,m.
$$

把变量按界缩放为 $x\leftarrow Xx,\ y\leftarrow Yy$，再取所有单项式系数组成整数格。PDF 用 $s=2$ 展示概念，此时只有 3 个移位，对应一个 $3\times3$ 小格；实际随仓库发布的 `solve.py` 调用

```python
for x0, y0 in modular_bivariate_homogeneous(
        x - h * y, p, m=5, t=5, X=X, Y=Y):
    if (x0 - h * y0) % p == 0:
        return ZZ(x0), ZZ(y0)
```

所以真正运行的是 6 个移位组成的 $6\times6$ 格。使用 `flatter` 或 Sage 的 LLL 约化后，短向量对应一个在真实小根处以整数意义消失的多项式 $H(x,y)$。

建格核心如下：

```python
shifts = []
for k in range(m + 1):
    shifts.append(y^(m-k) * F^k * p^max(t-k, 0))

L, monomials = create_lattice(
    polynomial_ring, shifts, [X, Y]
)
L = reduce_lattice(L)
polynomials = reconstruct_polynomials(
    L, F, p^t, monomials, [X, Y]
)
```

题目所参考的 [Improved Lattice-Based Attack on Mersenne Low Hamming Ratio Search Problem](https://eprint.iacr.org/2024/2080.pdf) 的核心作用，就是说明如何把 Mersenne 低汉明比搜索转成这种带界的格问题，并利用约化后的短关系恢复稀疏秘密。本文已经给出了本题真正使用的多项式、移位和恢复方式；链接保留用于查阅一般化攻击及其参数分析。

### 从齐次关系恢复 $f,g$

因为重建出的多项式是齐次的，可以令 $x=\tau y$、$y=1$，把二元方程化为关于比值 $\tau=x/y$ 的一元方程：

```python
t = var("t")
g = polynomials[0].subs(x=t*y).subs(y=1).simplify()
for root in solve(g == 0, t, domain=QQ):
    ratio = root.rhs()
    f = ZZ(ratio.numerator())
    g = ZZ(ratio.denominator())
    if (f - h * g) % p == 0:
        break
```

有理根的分子、分母给出候选 $(f,g)$。还应检查：

```python
assert (f - h * g) % p == 0
assert f.bit_count() == 10
assert g.bit_count() == 10
```

### 派生密钥并解密

恢复秘密后，严格复制题目中的大整数编码方式：

```python
secret = (f * g) % p
secret_bytes = secret.to_bytes((secret.bit_length() + 7) // 8, "big")
key = sha3_256(secret_bytes).digest()

raw = bytes.fromhex(ciphertext_hex)
iv, ct = raw[:16], raw[16:]
flag = unpad(AES.new(key, AES.MODE_CBC, iv).decrypt(ct), 16)
print(flag)
```

得到：

```text
L3ak{4jp2_n0t_s0_str0ng}
```

## 方法总结

本题的决定性弱点是两个秘密不仅稀疏，而且具有很小且严重不平衡的整数界。公钥同余 $f-hg\equiv0\pmod p$ 可直接写成齐次二元小根问题；缩放、构造模移位、LLL 约化后，再从齐次多项式的有理比值恢复分子和分母。阅读官方材料时要区分讲解参数与实际脚本参数：$s=2$、$3\times3$ 只是 PDF 的教学例子，仓库 solver 实际使用 $s=5$、$6\times6$。
