# rECp1cG

## 题目简述

rECp1cG 是一道椭圆曲线隐藏数问题。每次连接随机生成一个 1024 位素数 $p$ 和曲线：

$$
E: y^2=x^3+ax+b \pmod p
$$

服务端再随机选择起点 $P_0$ 和步长点 $G$，生成 21 个连续状态：

$$
P_i=P_0+iG,\qquad 0\le i\le 20
$$

公开的不是精确横坐标，而是带有有界误差的值：

$$
s_i=x(P_i)-r_i,\qquad |r_i|\le \Delta,\qquad \Delta=2^{451}
$$

玩家需要在 888 秒内恢复精确的 $x(P_0)$。回答正确后，服务端才返回 `key_tag` 和密文 `ct`；再结合完整的 $P_0$ 与公开曲线参数重建 SHA-256 密钥流，才能得到 flag。

本题的官方提示指向 [ECLCG/ECHNP 相关论文](https://eprint.iacr.org/2007/099.pdf)。论文的核心背景是：椭圆曲线状态的部分信息泄漏可转化为隐藏数/小根问题。但本题的误差达到 451 位，不能直接套用只泄漏少量低位时的简单恢复，需要先从曲线加法恒等式构造多变量模多项式，再用格约化和 2-adic 提升筛出真实小根。

## 解题过程

### 1. 明确已知量、未知量与最终目标

每次实例公开：

```text
p, a, b, Delta, G, states[0..20]
```

未知量包括起点 $P_0$ 以及 21 个误差 $r_i$。直接对所有误差建模规模过大，因此应利用这些点按固定步长 $G$ 连续排列的结构。

选择中点 $P_{10}$。对每个 $m=1,\ldots,10$，两侧点满足：

$$
P_{10+m}=P_{10}+mG
$$

$$
P_{10-m}=P_{10}-mG
$$

而 $mG$ 可由公开的 $G$ 直接计算。

### 2. 使用只含横坐标的加法恒等式

对曲线上的点 $P,Q$，存在只涉及横坐标的恒等式：

$$
\bigl(x(P+Q)+x(P-Q)\bigr)\bigl(x(P)-x(Q)\bigr)^2
=2\bigl(x(P)+x(Q)\bigr)\bigl(x(P)x(Q)+a\bigr)+4b
\pmod p
$$

令：

$$
X=s_{10}+x
$$

其中 $x=r_{10}$；再令：

$$
y_m=r_{10+m}+r_{10-m}
$$

则：

$$
x(P_{10+m})+x(P_{10-m})
=s_{10+m}+s_{10-m}+y_m
$$

将 $P=P_{10}$、$Q=mG$ 代入恒等式，便能对每个 $m$ 构造一个模 $p$ 的多项式：

$$
f_m(x,y_m)\equiv 0\pmod p
$$

一共得到 10 个方程，只剩 11 个小未知量：

$$
x,y_1,\ldots,y_{10}
$$

其界为：

$$
|x|\le 2^{451},\qquad |y_m|\le 2^{452}
$$

这种以中点配对的方式是降维关键：每个外侧误差只通过一对误差之和出现，不必同时恢复全部 21 个独立变量。

### 3. 构造多变量 Coppersmith 格

现在的问题是：多个多项式在模 $p$ 意义下有一个有界整数根。公开解法使用 XHS22 风格的多变量 Coppersmith 构造，实践参数为：

```text
center = 10
n      = 10
D      = 2
t      = 1
```

主要步骤如下：

1. 生成 $f_1,\ldots,f_{10}$ 的乘积、移位和乘以适当 $p$ 次幂后的多项式；
2. 收集所有单项式，建立系数矩阵；
3. 按变量上界缩放单项式，例如把 $x$ 按 $2^{451}$、$y_m$ 按 $2^{452}$ 缩放；
4. 对约 464 维的格执行约化；
5. 把短向量重新解释为在真实小根处应当很小、理想情况下在整数环上为 0 的多项式关系。

公开复现使用：

```sh
flatter -rhf 1.08
```

格约化是整个求解的主要耗时部分。服务只有 888 秒超时，实际部署时应提前准备好 Sage/格构造代码和 `flatter`，并限制线程、输入输出与后续筛选开销；不能在连接后才手工试参数。

### 4. 不盲信所有约化行

格约化后的每个短向量并不都能直接当作“在真实根处等于整数 0”的可靠方程。一部分关系只保证在模 $p^2$ 下消失，若把坏行全部交给代数求解或 Hensel 提升，往往提升几位后就出现矛盾。

固定使用某些行号也不稳健，因为每个远程实例的曲线和噪声都重新生成。更可靠的策略是保留较多候选关系，用真实根在 2-adic 意义下应持续满足它们这一性质动态选行。

### 5. 用 2-adic beam search 找到稳定分支

把 11 个未知整数从低位开始逐位恢复。第一层枚举所有奇偶模式：

$$
2^{11}=2048
$$

对每个候选根向量，统计约化多项式中有多少条在模 2 下成立；保留得分较高的一批状态。下一层给每个变量再追加一位，改在模 4 下评分，再继续到模 8、模 16：

$$
\bmod 2,\ \bmod 2^2,\ \bmod 2^3,\ldots
$$

伪代码如下：

```python
beam = all_parity_vectors(11)

for level in range(1, target_bits + 1):
    modulus = 1 << level
    candidates = []

    for state in beam:
        for next_bits in all_bit_vectors(11):
            lifted = state + (next_bits << (level - 1))
            score = count_relations_zero(lifted, modulus)
            candidates.append((score, lifted))

    beam = keep_best_and_diverse(candidates)
```

真实分支会在提升过程中持续满足大量关系，随机错误分支的有效行数则迅速下降。公开成功实例前几层最高分约为：

```text
level 3: 150
level 4: 142
level 5: 141
level 6: 140
```

当分支已经明显分离后，取仍在该分支上成立的关系做常规 Hensel 提升，直到覆盖误差界所需的约 454 位。若某一动态实例的最高分很快跌到阈值以下，应主动放弃连接并换一个实例，而不是在弱关系上耗尽 888 秒。

### 6. 从中点恢复起点

2-adic 求解给出中心误差 $r_{10}$，于是：

$$
x(P_{10})=s_{10}+r_{10}
$$

由曲线方程计算：

$$
v=x(P_{10})^3+a\,x(P_{10})+b\pmod p
$$

服务端生成 $p$ 时强制 $p\equiv 3\pmod 4$，所以平方根可写为：

$$
y=v^{(p+1)/4}\pmod p
$$

横坐标对应两个可能的点：

$$
(x,y),\qquad (x,-y\bmod p)
$$

分别计算：

$$
P_0=P_{10}-10G
$$

然后重新生成 $P_0+iG$，验证所有横坐标是否满足：

$$
|x(P_i)-s_i|\le \Delta
$$

只有一个符号分支会同时通过 21 个状态检查，这一步也恢复了密钥派生所需的 $y(P_0)$。

### 7. 重建密钥流并解密

提交正确的 $x(P_0)$ 后，服务端返回 `key_tag` 和 `ct`。源码按模数的字节长度，把以下六个整数以定长大端形式串联：

$$
a\parallel b\parallel x(G)\parallel y(G)\parallel x(P_0)\parallel y(P_0)
$$

密钥为：

```python
material = b"".join(
    value.to_bytes(size, "big")
    for value in (a, b, Gx, Gy, P0x, P0y)
)
key = sha256(material + b"|" + key_tag.encode()).digest()
```

密钥流按 32 字节分块生成：

```python
pad = b""
counter = 0

while len(pad) < len(ciphertext):
    pad += sha256(key + counter.to_bytes(4, "big")).digest()
    counter += 1

flag = bytes(c ^ k for c, k in zip(ciphertext, pad))
```

公开动态实例解得：

```text
r3ctf{3cHNP_ls_s0Oooooo_E2-f0r_thEmAN_U_4rE-c0PpERsmiTh_masteraaaaa0}
```

完整格基生成、`flatter` 接口和 beam search 工程实现可参考：[hax1ng 的 rECp1cG writeup](https://github.com/hax1ng/r3ctf-2026-writeups/blob/master/crypto/rECp1cG.md)。正文已根据本地 `challenge.py` 复核点生成、噪声符号、答案校验与密钥派生，外链用于保留高维格实现及一次成功实例的性能参数。

## 方法总结

本题的核心路线是：

```text
21 个连续 EC 点的噪声横坐标
  -> 以 P10 为中心配对 P10±mG
  -> x-only 加法恒等式
  -> 10 个模 p 多项式、11 个小未知量
  -> XHS22 风格多变量格
  -> flatter 约化得到候选整数关系
  -> 2-adic beam search 动态筛选稳定关系
  -> Hensel 提升恢复 r10
  -> 恢复 P10 与 P0 的正确 y 符号
  -> 提交 P0.x 并重建 SHA-256 密钥流
```

最重要的实践经验是：格约化输出不是可以无条件相信的方程集合。动态实例中，与其固定挑选某些短向量行，不如利用真实小根在连续 $2^k$ 模数下的稳定性来评分和筛选。对这类高维 ECHNP，格构造解决“产生足够多近似关系”的问题，而 2-adic 搜索解决“哪些关系真的能用”的问题，两者缺一不可。
