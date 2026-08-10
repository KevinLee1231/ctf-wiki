# HNP

## 题目简述

题目把 flag 的前 63 字节解释为整数 $m$，生成一个比 $m$ 多 8 位的强素数 $p$，并随机选取 32 个 $t_i<p$。每个样本先计算：

$$
x_i\equiv t_i m\pmod p,
$$

随后只泄露：

$$
r_i=x_i\bmod(2^{32}+1).
$$

已知 $p$、所有 $t_i$ 和 $r_i$，需要从每个模乘结果泄露的低模信息中恢复共同秘密 $m$。这是 Hidden Number Problem 的一种变形。

## 解题过程

### 建立近似整数关系

记 $D=2^{32}+1$。因为 $r_i$ 是 $x_i$ 对 $D$ 的余数，存在整数 $\ell_i$ 使：

$$
t_i m\equiv D\ell_i+r_i\pmod p.
$$

两边乘 $D^{-1}\bmod p$，可写成：

$$
t_iD^{-1}m-r_iD^{-1}\equiv\ell_i\pmod p.
$$

$m$ 比 $p$ 约少 8 位，而 $\ell_i$ 的规模可由题目参数估计为约 480 位。把 32 条同余式放入一个 34 维格，再额外加入用于表示 $m$ 和常数项的两行，就能让正确的 $m$ 对应一个短向量。

题目附件包含完整的 32 项 `t` 与 `res` 数组；前者是大量长整数，正文不重复粘贴，运行时从附件原样填入。官方 SageMath 解法整理如下：

```python
from Crypto.Util.number import long_to_bytes

p = ...
t = [...]      # 题目给出的 32 个 t_i
res = [...]    # 题目给出的 32 个 r_i

assert len(t) == len(res) == 32

D = 2^32 + 1
bound = 2^480
inv_D = inverse_mod(D, p)

M = matrix(QQ, 34, 34)
for i in range(32):
    M[i, i] = p
    M[-2, i] = t[i] * inv_D
    M[-1, i] = -res[i] * inv_D

# 两个缩放项分别承载秘密 m 和常数项。
M[-2, -2] = QQ(bound, p)
M[-1, -1] = bound

reduced = M.LLL()

# 官方数据中第二条约化向量对应目标关系。
m = ZZ(reduced[1, -2] / QQ(bound, p)) % p
plaintext = long_to_bytes(m)
print(plaintext)

# 用全部泄露值回代验证。
assert all((t[i] * m % p) % D == res[i] for i in range(32))
```

如果不同 SageMath 版本的 LLL 行顺序不同，不应固定只取 `reduced[1]`；可以遍历约化基的各行，按相同公式构造候选 $m$，再以 32 条泄露关系和 `hgame{` 前缀筛选。LLL 效果不稳定时，也可以用 BKZ 获得更强的格约化结果，但格的构造不变。

恢复结果为：

```text
hgame{H1dd3n_Numb3r_Pr0bl3m_has_diff3rent_s1tuati0n}
```

官方 PDF 给出 34 维格及恢复公式，但没有展示最终 flag；[参赛者题解](https://www.cnblogs.com/mumuhhh/p/18032304)给出了相同建模、BKZ 备选方法与最终结果，正文已经吸收这些关键信息。

## 方法总结

- 对模乘结果再取一个较小模数，会泄露关于同一秘密的近似线性关系；多组样本可转化为 HNP 格攻击。
- 应先显式写出 $t_i m\equiv D\ell_i+r_i\pmod p$，再决定格中各行、各列分别代表什么，避免只机械照搬矩阵。
- 缩放因子要匹配未知量的预期上界。本题用 $2^{480}$ 平衡 $\ell_i$、秘密项和常数项的尺度。
- 格约化基的行顺序不是稳定接口。可靠脚本应遍历候选并用全部 32 条泄露关系回代，而不是只相信固定索引。
