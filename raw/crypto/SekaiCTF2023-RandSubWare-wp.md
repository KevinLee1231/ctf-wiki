# RandSubWare

## 题目简述

服务每次生成一套随机的 5 轮 Substitution-Permutation Network（SPN）：

- block size 为 96 位；
- 每轮包含 16 个并行的 6 位 S-box；
- P-box 固定为均匀扩散各 S-box 输出位的置换；
- S-box 在会话开始时公开；
- 初始密钥为 96 位，轮密钥由前一轮密钥确定性派生；
- 最多允许加密 50000 个 block，随后要求提交原始密钥。

最后一轮只有 S-box 和轮密钥异或，没有 P-box。这使得可以按 6 位 S-box 独立猜测末轮子密钥，并用选择明文差分统计验证。官方解法依据 Howard Hey 的 [A Tutorial on Linear and Differential Cryptanalysis](http://www.cs.bc.edu/~straubin/crypto2017/heys.pdf)；本题还需要自行搜索适合当前随机 S-box/P-box 的高概率差分特征。

## 解题过程

### 建立差分分布表

对公开 S-box 枚举输入值 $x$ 和输入差分 $\Delta x$，统计：

$$
\operatorname{DDT}[\Delta x,\Delta y]
=\#\{x\mid S(x)\oplus S(x\oplus\Delta x)=\Delta y\}.
$$

```python
from collections import Counter

def difference_table(sbox):
    ddt = Counter()
    for dx in range(64):
        for x in range(64):
            dy = sbox[x] ^ sbox[x ^ dx]
            ddt[dx, dy] += 1
    return ddt
```

DDT 中计数越大，对应单个 S-box 差分传播的概率越高。

### 搜索三轮高概率差分特征

攻击最后一轮时，需要一个从明文输入到最后一轮 S-box 输入的三轮特征，并希望输出差分只激活一个 6 位分组。官方脚本用 Z3 为每一轮、每个 S-box 建立输入差分与输出差分变量：

```text
round r input differences
    -> DDT 允许的 S-box transition
    -> P-box bit constraints
    -> round r+1 input differences
```

约束包括：

1. 首轮输入差分不能全为零；
2. S-box 输入为零时输出也为零；
3. 非零输入、输出组合必须在 DDT 中出现；
4. P-box 后的每一位必须与下一轮对应输入位相等；
5. 三轮结束时只允许目标位置的 S-box 输入非零。

优化目标是提高各活动 S-box 的 DDT 计数乘积。对 16 个末轮位置分别求解，得到 16 条输入差分 $\Delta P_i$ 和预测的末轮输入差分 $\Delta U_i$。

### 猜测每个 6 位末轮子密钥

生成约 2941 个随机 96 位明文 $P_j$，一次性查询它们的密文 $C_j$。对位置 $i$，再查询：

$$
P_j' = P_j\oplus\Delta P_i.
$$

枚举该位置的 64 个末轮子密钥候选 $k$。对密文对应 6 位分组逆 S-box：

$$
\Delta U_j(k)
=S^{-1}(C_{j,i}\oplus k)
\oplus S^{-1}(C'_{j,i}\oplus k).
$$

统计 $\Delta U_j(k)$ 等于特征预测值的次数：

```python
scores = Counter()

for guess in range(64):
    for c1, c2 in zip(ciphertexts, paired_ciphertexts):
        u1 = inv_sbox[c1_block ^ guess]
        u2 = inv_sbox[c2_block ^ guess]
        scores[guess] += (u1 ^ u2) == expected_difference

last_round_key[position] = scores.most_common(1)[0][0]
```

正确猜测保留三轮差分偏差，错误猜测的计数更接近随机分布。对 16 个位置重复即可恢复完整 96 位末轮密钥。查询量为：

$$
2941\times(1+16)=49997,
$$

恰好低于 50000 block 配额。

### 逆向密钥扩展

轮密钥生成规则为：

$$
K_{r+1}=S^{\parallel16}(\operatorname{ROL}_{96}(K_r,7)).
$$

S-box 是置换且已公开，因此可逆：

$$
K_r=\operatorname{ROR}_{96}
\left((S^{-1})^{\parallel16}(K_{r+1}),7\right).
$$

从末轮密钥连续逆推 5 次，得到服务要求的初始密钥并提交。

## 方法总结

- 核心技巧：自动搜索随机 SPN 的高概率三轮差分特征，再按 S-box 独立统计恢复末轮密钥，最后逆向可逆 key schedule。
- 识别信号：SPN 轮数较少、S-box 公开、最后一轮不扩散、允许大量选择明文，并且轮密钥扩展完全可逆。
- 复用要点：差分密码分析不仅要有 DDT，还要找到能穿过具体 P-box 的有效特征。查询预算应按“基准明文集合 + 每个目标 S-box 的配对集合”计算；恢复末轮密钥后不要忘记题目可能要求的是初始密钥。
