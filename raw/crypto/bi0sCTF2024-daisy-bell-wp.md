# bi0sCTF 2024 - daisy_bell

## 题目简述

题目给出 2048 位 RSA 模数 $n=pq$、密文 $c$、素数 $p$ 的高位，以及 $q^{-1}\bmod p$ 的低 955 位。直接补全 $p$ 或枚举逆元高位都不可行，但这两段泄漏可以共同写成一个模 $n$ 的低根方程，再用二元 Coppersmith 恢复未知量。

## 解题过程

### 从 $p$ 的高位估计 $q$

已知的是

$$
p_{\mathrm{high}}=\left\lfloor\frac{p}{2^{545}}\right\rfloor.
$$

因此先用 $p\approx p_{\mathrm{high}}2^{545}$ 得到 $q$ 的高位近似：

$$
q_{\mathrm{high}}=\left\lfloor\frac{n}{p_{\mathrm{high}}2^{545}}\right\rfloor.
$$

把真实的 $q$ 写成

$$
q=q_{\mathrm{high}}+x,
$$

其中 $|x|<2^{545}$。常数项的具体对齐方式应与官方 solver 一致；本质要求只是把误差限制在约 545 位内。

### 构造二元低根方程

设题目泄漏的逆元为

$$
u=q^{-1}\bmod p=2^{955}y+u_{\mathrm{low}},
$$

未知高位 $y$ 约为 66 位。由逆元定义有

$$
uq\equiv1\pmod p.
$$

两边乘以 $q$，并利用 $n=pq$，得到关键关系

$$
q(uq-1)=uq^2-q\equiv0\pmod n.
$$

代入 $q=q_{\mathrm{high}}+x$ 与 $u=2^{955}y+u_{\mathrm{low}}$，构造二元多项式

$$
f(x,y)=\left(2^{955}y+u_{\mathrm{low}}\right)
\left(q_{\mathrm{high}}+x\right)^2-\left(q_{\mathrm{high}}+x\right).
$$

真实的 $(x,y)$ 是 $f(x,y)\equiv0\pmod n$ 的小根。官方解法取近似界

$$
|x|<2^{545},\qquad |y|<2^{1021-955},
$$

并以 $m=2$、总次数参数 $d=3$ 建立 Coppersmith 格。格约简后从短向量恢复两个整数多项式，求其公共根即可得到 $x$ 和 $y$。

求出 $x$ 后直接恢复因子：

```python
q = q_high + x
assert n % q == 0
p = n // q
phi = (p - 1) * (q - 1)
d = inverse(e, phi)
m = pow(c, d, n)
flag = long_to_bytes(m)
```

若低根实现返回多个候选，应逐一检查 `n % q == 0`；这是比仅检查多项式模值更可靠的最终验证。

## 方法总结

逆元低位泄漏之所以有用，是因为 $uq-1$ 含有因子 $p$。再乘一次 $q$ 就把未知模 $p$ 的关系提升成了模公开量 $n=pq$ 的关系。结合 $p$ 高位给出的 $q$ 近似，两个未知量都足够小，因而适合二元 Coppersmith。恢复任一素因子后，剩余步骤就是标准 RSA 解密。
