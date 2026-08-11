# good game spawn point

## 题目简述

服务生成 P-256 私钥 $x$ 并公布 $B=xG$，又生成 Paillier 密钥，却直接把 Paillier 的两个素因子 $p,q$ 输出。它最多接受五个攻击者给出的 Paillier 密文 $C$，返回一次乘法转加法（MtA）结果 $C^x\cdot E(\beta)$，以及一个名为 `zk schnorr` 的字典。

后者实际包含随机掩码的曲线点 $B_\beta=\beta G$，但没有返回 Schnorr 响应 $z$，也没有让客户端验证任何证明。结合可解密的 Paillier 响应，攻击者可以把每轮转成私钥的区间泄露，而非直接求完整 P-256 离散对数。

## 解题过程

### 恢复 Paillier 明文并写出群关系

由泄露的 $p,q$ 计算

$$
n=pq,\qquad \lambda=(p-1)(q-1),\qquad \mu=\lambda^{-1}\bmod n.
$$

对 `g=n+1` 的 Paillier 密文，官方 solver 使用

$$
D(c)=\frac{c^\lambda-1}{n}\mu\bmod n.
$$

选择 $C=E(k)=2^n(n+1)^k\bmod n^2$ 后，服务返回的密文解密为

$$
\alpha=D(C^xE(\beta))=kx+\beta\pmod n.
$$

令 $s$ 满足 $kx+\beta=\alpha+ns$。服务同时泄露 $B_\beta=\beta G$，故可计算

$$
H=n^{-1}(kB+B_\beta-\alpha G)=sG.
$$

这一步只使用题目给出的公钥、`beta_pub` 和可解密的响应；`hash` 与 `r_pub` 不参与恢复。关键缺陷正是把本应仅用于零知识证明内部的 $\beta G$ 公开，却又没有提供或校验完整证明。

### 以可控 $k$ 缩小私钥区间

初始设 $x\in[0,\operatorname{order})$。官方脚本取

$$
k_i=\left\lfloor\frac{2^{43i}n}{\operatorname{order}}\right\rfloor.
$$

若当前候选区间为 $[L,U]$，则 $s$ 落在约为 $[k_iL/n,k_iU/n]$ 的窄区间；脚本在两端各留 1 的余量以覆盖取整和 $\beta<n$。在这个范围内对 $H=sG$ 做 BSGS，得到 $s$ 后反推新边界：

$$
L\leftarrow\left\lfloor\frac{(s-1)n}{k_i}\right\rfloor,
\qquad
U\leftarrow\left\lceil\frac{(s+1)n}{k_i}\right\rceil.
$$

每轮约削去 43 bit；五轮后，候选宽度已小于 $2^{43}$，最后直接在 $B=xG$ 的剩余区间做一次 BSGS。

以下为官方 `solve.py` 的变量关系骨架；`query`、`parse_beta_pub` 与 `bounded_bsgs` 分别对应一轮网络收发、曲线点反序列化和带上下界的 BSGS 实现。

```python
while (high - low).bit_length() >= 43:
    k = (2 ** (43 * round_no) * n) // order
    c = pow(2, n, n * n) * pow(n + 1, k, n * n) % (n * n)
    alpha = paillier_decrypt(query(c), lam, mu, n)
    beta_point = parse_beta_pub()

    H = (k * B + beta_point - alpha * G) * inverse_mod(n, order)
    s = bounded_bsgs(G, H, floor(k * low / n) - 1,
                      ceil(k * high / n) + 1)
    low = floor((s - 1) * n / k)
    high = ceil((s + 1) * n / k)

x = low + bounded_bsgs(G, B - low * G, 0, high - low)
```

### 验证

官方 `solve.py` 最终将恢复的 `secret_calc` 送入 `guess secret:`；服务以整数相等比较决定是否输出 flag。题目配置给出的验证材料为 `DUCTF{g00d_gam3_5pawnling_533_y0u_in_th3_n3xt_chall3ng3}`。本归档仅静态核对了源码、参数和官方 solver，没有运行 5 次 BSGS 查询。

## 方法总结

- 核心技巧：同态加密响应泄露 $kx+\beta$，同时泄露 $\beta G$ 时，可消去掩码并把秘密标量映射为一个小范围离散对数。
- 识别信号：协议同时输出 Paillier 私钥参数、可控密文的 MtA 结果，以及掩码的 EC 承诺/公钥时，应先写出明文和曲线群里的同一线性关系。
- 复用要点：不必暴力完整曲线阶；将 $k$ 选为与 $n/\operatorname{order}$ 成比例的幂，可把商 $s$ 变成一次区间测量，再用 BSGS 和整数上下界逐轮收缩。
