# r3system-revenge

## 题目简述

revenge 版去掉了四轮 Keccak PoW，但连接后必须先提交基础题给出的 `secret_flag_part`。进入系统后的令牌、120 次多项式 PRNG、P-256 ECDH 和 AES-ECB 流程与 `r3system` 相同。

关键修复是把系统私钥从 Python `random.randint()` 改为：

```python
self.x = int(sha256(os.urandom(32)).hexdigest(), 16) % MOD
```

因此不能再利用全局 Mersenne Twister 状态预测 $x$。预期路线是收集连续令牌，用类 Vandermonde 行列式消去未知 PRNG 系数，直接建立只含 $x$ 的有限域多项式。

## 解题过程

### 收集 123 个连续令牌

对不同用户名 $u_i$ 注册账户，服务返回：

$$
t_i=x(k_i+h_i)^{-1}\pmod P,\qquad
h_i=\operatorname{SHA256}(u_i).
$$

所以：

$$
k_i=xt_i^{-1}-h_i\pmod P.
$$

`Users.register()` 的上限恰好允许取得 123 个选手账户令牌。用户名必须互不相同，并按接收顺序保存用户名和 token；中间不要触发其他会调用 `RNG.next()` 的路径，否则连续状态的索引会错位。

### 用行列式约束恢复 x

`RandomNG` 有 121 个系数：

$$
k_{i+1}=a_{120}k_i^{120}+\cdots+a_1k_i+a_0\pmod P.
$$

对 123 个状态建立 122 条递推关系。令：

$$
\mathcal M=
\begin{pmatrix}
k_1&k_0^{120}&k_0^{119}&\cdots&k_0&1\\
k_2&k_1^{120}&k_1^{119}&\cdots&k_1&1\\
\vdots&\vdots&\vdots&&\vdots&\vdots\\
k_{122}&k_{121}^{120}&k_{121}^{119}&\cdots&k_{121}&1
\end{pmatrix}.
$$

第一列等于其余列按 $(a_{120},\ldots,a_0)$ 的线性组合，所以 $\det\mathcal M=0$。沿第一列展开，可把余子式写成 Vandermonde 乘积：

$$
\det\mathcal M=
\sum_{i=0}^{121}(-1)^i k_{i+1}
\prod_{\substack{0\le r<s\le121\\r,s\ne i}}
(k_s-k_r).
$$

逐个代入 $k_i=xt_i^{-1}-h_i$ 后，得到 $F(x)=0\pmod P$。实现时应直接累计每个 Vandermonde 乘积，避免让 CAS 展开一个 $122\times122$ 的符号行列式；作者给出的优化把不可接受的递归多项式计算降到了可在比赛时限内完成的规模。

$F$ 的次数仍然很高。由于有限域 $\mathbb F_P$ 中任意元素都满足 $X^P-X=0$，先计算模 $F$ 的 $X^P-X$，再求：

$$
G(X)=\gcd(F(X),X^P-X).
$$

最后分解低次的 $G$ 或调用 `roots()`，得到所有域内候选 $x$。

### 用公开 ECDH 数据验证并取 flag

公开频道给出了 Alice 与 Bob 的 P-256 公钥，以及 Alice 发给 Bob 的加密 token。对每个 $x$ 候选按源码派生 Alice 私钥：

$$
d_A=\operatorname{SHA256}(\operatorname{str}(x))
\bmod n_{\text{P-256}},
$$

计算 $d_AG$ 并与 Alice 公钥对比。唯一匹配者即为正确 $x$。

随后：

$$
S=d_AQ_B,\qquad
K=\operatorname{MD5}(\operatorname{str}(S)).
$$

用 $K$ 作为 AES-ECB 密钥解密公开频道中的密文。明文开头 32 字节是 `FLAG_TOKEN`；revenge 的菜单不再要求 `PoW_KinG`，直接选择 `Guess Flag Token` 并提交即可。

## 方法总结

revenge 真正修复的是基础版的可预测随机私钥，而不是令牌方程本身。只要能取得 123 个连续输出，每个状态仍是同一未知 $x$ 的一次函数；多项式递推又强迫类 Vandermonde 矩阵奇异，从而留下一个只关于 $x$ 的代数约束。

完整的余子式加速、有限域求根方法与性能数据见作者的 [r3system / r3system-revenge 说明](https://github.com/D33BaT0/my-ctf-challenges/tree/main/r3ctf%202025-crypto/r3system)。基础题的 Keccak 碰撞不属于本题主体，但其 `secret_flag_part` 是连接 revenge 的必要前置输入。
