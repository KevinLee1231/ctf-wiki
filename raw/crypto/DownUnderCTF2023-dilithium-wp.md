# DownUnderCTF 2023 dilithium plusminus Writeup

## 题目简述

题目基于 Dilithium2 参考实现，但把秘密向量维度参数 `L` 从 4 改为 2，并篡改了签名阶段对 $\mathbf z$ 的拒绝采样。服务提供公钥和 19091 个签名，要求为消息 `dilithium crystals` 伪造签名。

## 解题过程

Dilithium 签名中的主要关系为：

$$
\mathbf z=\mathbf y+c\mathbf s_1.
$$

其中 $\mathbf y$ 在接近 $[-\gamma_1,\gamma_1]$ 的范围均匀取样，挑战多项式 $c$ 只有 $\tau=39$ 个系数为 $\pm1$，而秘密 $\mathbf s_1$ 的每个系数都位于 $[-2,2]$。

标准实现会拒绝范数过大的 $\mathbf z$，避免签名分布泄露秘密。补丁却要求至少有一个系数达到 $\gamma_1-2$，同时只拒绝绝对值大于 $\gamma_1$ 的结果。因此每个签名都刻意保留了边界事件。

若某个边界系数 $z\ge0$，由 $z=y+c\mathbf s_1$ 和 $y\le\gamma_1$ 可得：

$$
z-\gamma_1\le c\mathbf s_1.
$$

若 $z<0$，则可得：

$$
c\mathbf s_1\le\gamma_1-|z|-1.
$$

结合原本的 $[-\beta,\beta]$ 范围，每个极值系数都能转成一个线性不等式。官方求解器只选取恰有一个目标系数等于 $\gamma_1-2$、$\gamma_1-1$ 或 $\gamma_1$ 的签名，构造 CP-SAT 约束：

```python
if is_negative:
    model.AddLinearConstraint(c_times_s1, -BETA, GAMMA1 - target - 1)
else:
    model.AddLinearConstraint(c_times_s1, target - GAMMA1, BETA)
```

打包格式不能正常表示 $-\gamma_1$，它会回绕成 $+\gamma_1$。因此当解包值为 $\gamma_1$ 时，再用公开密钥验证原签名：验证失败说明签名时实际使用的是负边界，验证成功则是正边界。

对 `L=2` 的两个多项式分别建立 256 个整数变量，每个变量范围为 $[-2,2]$。约 9500 条有效不等式足以让 OR-Tools 恢复完整 $\mathbf s_1$。

得到 $\mathbf s_1$ 后，不需要恢复全部私钥即可伪造。随机选择 $\mathbf y$，计算 $\mathbf w=A\mathbf y$ 和挑战 $c$，再令 $\mathbf z=\mathbf y+c\mathbf s_1$。利用公钥中的 $t_1$ 计算验证方看到的高位，并构造提示向量 $h$：

```python
mu = shake_256(shake_256(pk_bytes).digest(32) + target_message).digest(64)

while True:
    y = polyvecl_uniform_gamma1(urandom(64))
    w = compute_A_mul_v(Ahat, y)
    w1, _ = polyveck_decompose(w)
    c = shake_256(mu + polyveck_pack_w1(w1)).digest(32)
    cp = poly_challenge(c)
    z = y + cp * s1

    verifier_w1 = compute_A_mul_v(Ahat, z) - cp * t1 * 2**D
    verifier_w1, _ = polyveck_decompose(verifier_w1)
    h = verifier_w1 - w1
    if sum(coeff == 1 for poly in h for coeff in poly) > OMEGA:
        continue

    signed_message = pack_sig(c, z, h) + target_message
    if verify(signed_message, pk_bytes) is not False:
        break
```

提交有效签名后得到：

```text
DUCTF{1n_r34l_l1fe_dilithium_is_actu4lly_a_m0l3cule_ce75f66577ef45e17edde8a5b4}
```

## 方法总结

Fiat-Shamir with Aborts 中的“Aborts”是安全机制，不是性能细节。把拒绝条件反向修改为只保留边界样本，会把 $c\mathbf s_1$ 的符号和界限逐步泄漏成线性约束。大量签名、很小的秘密系数域和被缩短的 `L` 共同使 CP-SAT 能直接恢复秘密向量。
