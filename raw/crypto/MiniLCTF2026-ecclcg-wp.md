# ecclcg

## 题目简述

题目在未知曲线 $E:y^2=x^3+ax+b\pmod p$ 上生成 EC-LCG：$P_i=P_{i-1}+G$。输出的 $G$ 与连续 $21$ 个 $P_i$ 坐标均被减去绝对值至多 $\Delta=2^{192}$ 的误差，而 $p$ 为 1024-bit；另给出以 $a,b,G,P_0$ 派生的 AES-CTR 密钥材料。关键泄露是连续点的加法关系仍在小误差下保留模 $p$ 的近似同余。

## 解题过程

### 从三点共线得到小量同余

记公开坐标为 $\tilde G=(\gamma_x,\gamma_y)$、$\tilde P_i=(\alpha_i,\beta_i)$，真实坐标为

$$
G=(\gamma_x+h_x,\gamma_y+h_y),\qquad P_i=(\alpha_i+e_i,\beta_i+f_i).
$$

由 $P_i=P_{i-1}+G$ 可知 $P_{i-1},G,-P_i$ 共线，因此

$$
(y_{i-1}+y_i)(x_{i-1}-G_x)-(y_{i-1}-G_y)(x_{i-1}-x_i)\equiv0\pmod p.
$$

定义

$$
M=h_x-e_0,\ E_i=e_{i-1}-e_i,\ F_i=f_{i-1}+f_i,\ N_i=f_{i-1}-h_y,
$$

并令 $m_i=M+\sum_{j<i}E_j$。将坐标展开后，每条转移给出

$$
b_im_i+q_iE_i+p_iF_i+a_iN_i+\Sigma_i\equiv-(b_ip_i+a_iq_i)\pmod p,
$$

其中 $b_i=\beta_{i-1}+\beta_i$、$p_i=\gamma_x-\alpha_{i-1}$、$a_i=\alpha_{i-1}-\alpha_i$、$q_i=\beta_{i-1}-\gamma_y$，以及人为引入的

$$
\Sigma_i=m_iF_i+E_iN_i.
$$

这样非线性的椭圆曲线加法被拆成线性同余加一个可后验检查的小乘积项。

### 格恢复误差

以 $(M,E_i,F_i,N_i,\Sigma_i)$ 为未知量，把 $20$ 条同余写成整数格。为平衡量级，将前四类量缩放为 $\Delta M,\Delta E_i,\Delta F_i,\Delta N_i$，并在格中加入它们均为 $\Delta$ 倍数的约束。对格基进行 HNF/LLL（必要时 BKZ）约简，再对同余常数目标执行 Babai 近似 CVP。

候选必须先能按 $\Delta$ 整除，随后逐项检查

$$
\Sigma_i=m_iF_i+E_iN_i.
$$

再利用

$$
h_y=\frac{F_i-N_i-N_{i+1}}2
$$

在所有 $i$ 上一致地恢复 $h_y,f_i$。这会淘汰仅满足线性格约束的假候选。

### 确定剩余平移、曲线与密钥

$x$ 方向仍有 $e_0=t$ 的平移自由度：$h_x=M+t$，$e_i=t-\sum_{j\le i}E_j$。将第一条点加法的 $x$ 坐标关系写为

$$
(Y_0-G_y)^2-(X_1+X_0+G_x)(X_0-G_x)^2\equiv0\pmod p,
$$

求该一元式的根 $t$，并重新检查所有误差界。由恢复的 $G,P_0$ 求

$$
a=\frac{(G_y^2-G_x^3)-(P_{0,y}^2-P_{0,x}^3)}{G_x-P_{0,x}}\pmod p,
\qquad b=G_y^2-G_x^3-aG_x\pmod p.
$$

最终验证每个点都在曲线上，且从 $P_0$ 连续加 $G$ 恰好重现所有状态点。仅在这一步通过后，才按题目顺序把 $a,b,G_x,G_y,P_{0,x},P_{0,y}$ 固定为 $128$ 字节、拼接 `|key_tag`、取 SHA-256 前 16 字节并 AES-CTR 解密。

## 方法总结

- 核心技巧：将相邻 EC-LCG 状态的共线条件展开为小误差的线性模同余，再用 CVP 找短向量。
- 识别信号：大素数模、少量连续椭圆曲线状态和远小于模数的坐标扰动，通常意味着格攻击而非直接猜测曲线参数。
- 复用要点：乘积变量、误差范围、曲线方程和完整状态转移构成逐层验证链；固定宽度编码同样是密钥派生的一部分。
