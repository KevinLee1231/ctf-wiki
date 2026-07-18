# SUCTF2026-RSA

## 题目简述

题目生成两个 512 位素数 $p,q$，令 $N=pq$，并直接选择一个约 337 位的素数私钥指数：

```python
bits = 1024
delta0 = 0.33
gamma = 0.39
d = getPrime(int(bits * delta0))
e = inverse(d, (p - 1) * (q - 1))
S = (p + q) - (p + q) % 2**int(bits * gamma)
```

因此公开的 $S$ 是 $p+q$ 清零低 399 位后的结果。记

$$
p+q=S+w,qquad 0\le w<2^{399}.
$$

附件中另有 `delta=0.08`，但它没有参与任何计算；真正控制 $d$ 的是 `delta0=0.33`。题目同时给出 $N,e,c,S$，目标是利用小 $d$ 与近似素数和恢复私钥。

官方设计参考论文 [Improving RSA Cryptanalysis: Combining Continued Fractions and Coppersmith's Techniques](https://eprint.iacr.org/2025/1281)。论文把连分数提供的关键关系与 Coppersmith 结合，并研究了已知 $p+q$ 近似值时更大的小私钥指数攻击范围；其主结果以 $\alpha=\log_Ne$、$\gamma=\log_N|p+q-S|$ 表示可攻击界。作者的 [RSA_CFL](https://github.com/MengceZheng/RSA_CFL) 仓库提供相应 Sage 实现及参数化入口。

不过出题参数没有卡到必须使用论文三元模型：把 RSA 密钥方程直接写成关于 $k$ 与 $w$ 的二元小根，便能更快求解。这是官方 WP 明确记录的非预期简化路线，也是本题实际采用的主解。

## 解题过程

### 1. 官方预期：连分数关系转三元小根

RSA 密钥方程为

$$
ed-k(p-1)(q-1)=1.
$$

论文方法取 $e/N$ 的一对相邻连分数逼近

$$
\frac{p_{r-1}}{q_{r-1}},\qquad\frac{p_r}{q_r},
$$

并将小私钥乘子写为

$$
d=u q_r+v q_{r-1},qquad k=u p_r+v p_{r-1}.
$$

再代入

$$
\varphi(N)=N+1-(p+q)=N+1-S-w,
$$

即可在模 $e q_r$ 下整理出

$$
f(w,u,v)=wu+a_1wv+a_2u+a_3v+a_4\equiv0\pmod{e q_r},
$$

其中 $a_i$ 都由 $N,e,S$ 和两组连分数逼近确定。对有界未知量 $(w,u,v)$ 构造移位多项式格，LLL 后提取代数独立的小系数多项式，便可用三元 Coppersmith 恢复根。论文方法的价值在于连分数先把 $d,k$ 投影到较小的 $u,v$ 上，从而扩大可攻击参数区间。

本题可以调用作者仓库的实现，但官方记录其在生成数据上约需数分钟；由于下面的二元界已经足够宽，没有必要承担三元格的额外成本。

### 2. 实际解法：直接构造二元模方程

令

$$
A=N-S+1.
$$

则

$$
\varphi(N)=A-w.
$$

存在整数

$$
k=\frac{ed-1}{\varphi(N)}
$$

满足

$$
1+k(A-w)\equiv0\pmod e.
$$

因为 $0<e<\varphi(N)$，有 $k<d<2^{338}$；同时 $w<2^{399}$。令

$$
x=k,qquad y=-w,
$$

得到二元小根：

$$
f(x,y)=1+x(A+y)\equiv0\pmod e,
$$

$$
|x|<X=2^{338},qquad |y|<Y=2^{399}.
$$

其乘积界约为 $2^{737}$，明显小于约 1024 位的模数 $e$，本题参数给了足够的格攻击余量。

### 3. 构造 Herrmann–May 风格二元格

官方 `exp.py` 引入辅助变量

$$
u=xy+1,qquad |u|<U=XY+1,
$$

并在商环 $\mathbb Z[u,x,y]/(xy+1-u)$ 中处理 `f`。这样所有 $xy$ 项都被线性化成 $u-1$，可避免单项式集合无控制膨胀。

利用参数 `M=3, T=1` 构造两类移位：

```text
x^i * e^(M-k) * f^k
y^j * e^(M-k) * f^k    # 在商环中约化
```

对每个单项式按界 $(U,X,Y)$ 缩放系数形成整数格，LLL 后从前若干短向量重建在真根处取整值 0 的多项式。总 PDF 另写了一套 39 维手工移位矩阵，并记录了循环边界、变量选择导致方阵不匹配的问题；官方仓库中的商环构造更短，也与最终验证逻辑一致，优先采用。

### 4. 用 resultant 提根并严格验真

将约减后多项式中的 $u$ 替换回 $xy+1$，取两条非平凡多项式，对 $y$ 或 $x$ 求 resultant 消元。遍历前若干短向量组合，取得整数根候选 $(k,-w)$ 后依次检查：

```python
phi = A - w
assert (1 + k * phi) % e == 0
d = (1 + k * phi) // e

s = N - phi + 1                 # p + q
Delta = s * s - 4 * N
assert is_square(Delta)
p = (s + isqrt(Delta)) // 2
q = (s - isqrt(Delta)) // 2
assert p * q == N
```

直接用 `inverse_mod(e, phi)` 虽可能得到相同 $d$，但不能代替上述因子判定；resultant 可能给出无关整数根，必须用判别式完全平方和 $pq=N$ 过滤。

最后解密：

```python
m = pow(c, d, N)
flag = long_to_bytes(m)
```

得到：

```text
SUCTF{congratulation_you_know_small_d_with_hint_factor}
```

### 5. 两条路线的边界

- 连分数 + 三元 Coppersmith 是题目原定知识点，也是在更紧参数下应使用的通用方法。
- 本实例的 $d$ 只有约 $N^{0.33}$，且 $p+q$ 的误差仅约 $N^{0.39}$；二元方程中的 $k$、$w$ 已足够小，所以无需先用连分数把它们改写成 $u,v$。
- 这不等于经典 Boneh–Durfee“只给 $N,e$”场景；成功依赖额外泄露的 $p+q$ 高位。更准确的描述是带素数和近似值的二元 Coppersmith/Herrmann–May 小根攻击。

## 方法总结

- 先确认实际参数：`delta0=0.33` 决定 337 位 $d$，`gamma=0.39` 决定 399 位未知素数和低位；不要被未使用的 `delta` 误导。
- 官方预期用相邻连分数把 $d,k$ 表成两个小系数，再求三元小根；论文和作者仓库链接已保留并在正文中概括其用途。
- 本实例可直接利用 $1+k(N-S+1-w)\equiv0\pmod e$，以 $k<2^{338}$、$w<2^{399}$ 构造二元格，速度更快。
- 小根候选必须通过 RSA 方程、判别式平方、`p*q==N` 和最终明文格式四层校验。
