# r3system

## 题目简述

本题由两个连续阶段组成。第一阶段要求在 600 秒内提交 22 对四轮 Keccak-224 碰撞，而且每一对的第一个消息都必须以字节 `r` 开头；只有全部满足，`PoW_KinG` 才会置为真。

第二阶段是一套令牌与 ECDH 消息系统。每个注册令牌为：

$$
t_i=x(k_i+H(u_i))^{-1}\pmod P,
$$

其中 $x$ 同时决定 Alice 的 ECDH 私钥，$k_i$ 来自一个 120 次多项式状态递推。注册接口最多提供 123 个连续令牌。公开频道中则包含 Alice、Bob 的公钥和用双方共享密钥加密的 32 字节 `FLAG_TOKEN`。恢复 $x$ 后即可解密 token，再通过菜单换取 flag。

## 解题过程

### 构造 22 对四轮 Keccak-224 碰撞

题目使用的是 reduced-round Keccak，而不是标准库中的完整 Keccak-224。一轮置换可写为：

$$
R=\iota\circ\chi\circ\pi\circ\rho\circ\theta.
$$

其中 $\theta,\rho,\pi$ 都是线性的，只有 $\chi$ 是非线性的 5 位 S-box。官方解法依据 Keccak-224/256 的 Target Difference Algorithm（TDA），分成两步：

1. 差分阶段：求初始差分 $\Delta_i=M_1\oplus M_2$，要求 padding 与 capacity 位上的差分为零，并令线性层后的差分符合 $\chi$ 的差分分布；结果是可行差分的仿射子空间。
2. 取值阶段：固定 $\Delta_i$ 后，为第一条消息求仿射空间，使每个 $\chi$ S-box 的输入对都产生目标输出差分。逆线性层后可写成：

$$
M_1=M_0\oplus\sum_j c_jv_j,\qquad
M_2=M_1\oplus\Delta_i.
$$

在该空间中枚举核向量组合，逐对执行题目自带的四轮 `Keccak224()` 验证。服务还检查 `x[0] == 114`，因此要把首字节等于 `r` 作为八个 GF(2) 线性约束加入消息空间，而不是生成碰撞后再修改首字节。

从空间中选取 22 个不同系数组合即可生成 22 对互不重复的碰撞。提交时使用十六进制，每条消息不能超过服务截取的 1488 个十六进制字符。

### 用连续令牌消去多项式系数

令用户名哈希为 $h_i=H(u_i)$，已知令牌为 $t_i$，则每个 PRNG 状态都可写成未知 $x$ 的函数：

$$
k_i=xt_i^{-1}-h_i\pmod P.
$$

源码中递推为：

$$
k_{i+1}=\sum_{j=0}^{120}a_jk_i^j\pmod P.
$$

注册 123 个不同账户，收集 123 个连续令牌。对前 122 次转移构造 $122\times122$ 矩阵：

$$
\mathcal M=
\begin{pmatrix}
k_1&k_0^{120}&k_0^{119}&\cdots&k_0&1\\
k_2&k_1^{120}&k_1^{119}&\cdots&k_1&1\\
\vdots&\vdots&\vdots&&\vdots&\vdots\\
k_{122}&k_{121}^{120}&k_{121}^{119}&\cdots&k_{121}&1
\end{pmatrix}.
$$

第一列必然是后 121 列的线性组合，因此：

$$
\det(\mathcal M)=0\pmod P.
$$

这一步消去了未知的全部 $a_j$。直接求符号行列式代价过高，可以沿第一列展开；每个余子式都是 Vandermonde 行列式：

$$
\det(C_i)=
\prod_{\substack{0\le r<s\le121\\r,s\ne i}}
(k_s-k_r).
$$

把 $k_i=xt_i^{-1}-h_i$ 代入并清除已知分母，得到只含 $x$ 的多项式 $F(x)$。在有限域 $\mathbb F_P$ 中计算：

$$
G(X)=\gcd\left(F(X),X^P-X\right),
$$

即可把高次多项式限制到域内的一次因子，枚举 `G.roots()` 得到少量候选。

### 验证 x 并解密 FLAG_TOKEN

Alice 的 ECDH 私钥不是 $x$ 本身，而是：

$$
d_A=\operatorname{SHA256}(\operatorname{str}(x))\bmod n_{\text{P-256}}.
$$

对每个根计算 $d_AG$，与公开频道中的 Alice 公钥比较，即可确定唯一的 $x$。再用 Bob 公钥 $Q_B$ 计算：

$$
S=d_AQ_B,\qquad
K=\operatorname{MD5}(\operatorname{str}(S)).
$$

源码使用 AES-ECB，以这个 16 字节 MD5 值为密钥加密 `FLAG_TOKEN`。从公开频道取出 Alice 发给 Bob 的十六进制密文并解密，前 32 字节就是 token。回到未登录菜单选择 `Guess Flag Token`，提交 token；由于第一阶段已经成为 `PoW_KinG`，服务会输出 flag。

## 方法总结

本题要求分别利用哈希差分与相关状态：TDA 把四轮 Keccak 碰撞搜索压缩到 GF(2) 仿射空间，首字节限制也能作为线性条件并入；令牌阶段则把每个隐藏状态表示成同一个私钥 $x$ 的函数，再以类 Vandermonde 行列式消去 121 个未知递推系数。

作者仓库的 [r3system / r3system-revenge 说明](https://github.com/D33BaT0/my-ctf-challenges/tree/main/r3ctf%202025-crypto/r3system) 给出了 TDA 的两阶段构造、行列式展开优化及求根方式。官方同时记录了基础版中 Python `random.randint` 带来的非预期简化，但上述矩阵方法不依赖该实现失误，也适用于修复后的 revenge。
