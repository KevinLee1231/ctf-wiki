# r3tauq

## 题目简述

题目生成三个 256 位素数 $p,q,r$，以及低 128 位为零的 384 位整数 $x,y$，然后在：

$$
\operatorname{QuaternionAlgebra}(\mathbb Z_{pq},-x,-y)
$$

中计算：

$$
(x+y,\ p+x,\ q+y,\ r)^S,
$$

其中 `secret` 是 77 个英文字母，$S$ 是把 `secret` 重复 777 次后按大端整数解释的结果。服务只输出 $n=pq$ 和幂运算所得四个四元数系数，要求在三次机会内提交完整 `secret`。

解法分为三层：从四元数幂的三个虚部恢复 $p,q,x,y$；把四元数离散对数降到有限域离散对数；最后利用 77 个 ASCII 字节都很小的约束，从模意义下的指数恢复原字符串。

## 解题过程

### 用共同倍数关系恢复模数因子

四元数 $Q=a+bi+cj+dk$ 的幂具有一个重要性质：对任意正整数 $t$，$Q^t$ 的三个虚部仍是原虚部的同一个标量倍数。若输出为 $(A,B,C,D)$，则存在同一个 $L$ 使：

$$
B=L(p+x),\quad C=L(q+y),\quad D=Lr\pmod n.
$$

其中 $p+x$、$q+y$ 约 384 位，$r$ 约 256 位，均显著短于未知倍数造成的普通 512 位模剩余。构造格：

$$
\mathcal L=
\begin{pmatrix}
B&C&D2^{128}\\
n&0&0\\
0&n&0\\
0&0&n2^{128}
\end{pmatrix}.
$$

第三列乘 $2^{128}$ 是为了平衡三维尺度。LLL 约化后，可从最短向量取得符号不定的：

$$
(p+x,\ q+y,\ r2^{128}).
$$

由于 $x,y$ 本身都左移了 128 位：

$$
p+x\equiv p\pmod{2^{128}},\qquad
q+y\equiv q\pmod{2^{128}}.
$$

这直接泄露了 $p,q$ 的低 128 位。单变量 Coppersmith 在理论边界附近不够稳定，可再枚举相邻的 8 个未知位，把已知低位扩展到 136 位，再对：

$$
f(z)=2^{136}z+p_{\mathrm{low},136}
$$

以 $n$ 的未知因子 $p$ 为模寻找小根。命中后用 `gcd(f(z), n)` 得到 $p$，继而算出 $q=n/p$；最后由短向量减去 $p,q$ 得到 $x,y$。

### 把四元数 DLP 降到有限域

对底环中的四元数定义约化范数：

$$
\operatorname{Norm}(a+bi+cj+dk)
=a^2+xb^2+yc^2+xyd^2.
$$

它满足乘法同态，因此：

$$
\operatorname{Norm}(Q^S)=\operatorname{Norm}(Q)^S.
$$

分别模 $p$、模 $q$ 计算底数四元数和输出四元数的范数，得到两个普通有限域离散对数：

$$
H_p=G_p^S\pmod p,\qquad
H_q=G_q^S\pmod q.
$$

先分解 $p-1$ 与 $q-1$，小因子用 Pohlig–Hellman 处理；题目实例中还各有约 200 位的大素数阶子群，需要用 CADO-NFS 的 DLP 模式求对数。将各子群结果合并，得到：

$$
S\equiv S_0\pmod m,\qquad
m=\operatorname{lcm}(\operatorname{ord}(G_p),\operatorname{ord}(G_q)).
$$

### 从模指数恢复 77 字节 secret

令 $z=\operatorname{bytes\_to\_long}(\texttt{secret})$，每次重复占 $616=77\cdot8$ 位，则：

$$
S=zT,\qquad
T=\sum_{i=0}^{776}2^{616i}.
$$

于是：

$$
z\equiv z_0=S_0T^{-1}\pmod m.
$$

$z$ 有 616 位，而 $m$ 约 508 位，仅靠同余不能唯一确定。但每个字节只能是 `A-Z` 或 `a-z`。把字节写成 $c_i=93+d_i$，则每个 $d_i$ 都是绝对值不超过 29 的小整数。将：

$$
\sum_{i=0}^{76}c_i2^{8(76-i)}-z_0=km
$$

嵌入格中，对同余列放大后使用 BKZ/LLL，短向量会给出全部小偏移 $d_i$。枚举约化基中的符号和短组合，筛选 77 个字节均为英文字母的候选，再用原四元数幂验证；通过后把字符串提交服务即可。

## 方法总结

本题把四元数结构、格攻击和有限域 DLP 串成了一条长链。三个关键降维分别是：四元数幂的虚部共享标量，使隐藏底数落入低维短向量；约化范数把非交换代数中的幂映射为有限域乘法；重复字符串结构与 ASCII 字节范围又把 616 位未知量变成可由格恢复的小偏移问题。

完整 Sage 求解器、CADO-NFS 参数示例和字节恢复格可参照 [tl2cents 的 r3tauq 题解仓库](https://github.com/tl2cents/CTF-Writeups/tree/master/2025/R3CTF/r3tauq)。正文使用的是本仓库实际部署版本：最终目标是还原并提交 `secret`，不依赖额外的 AES 密文。
