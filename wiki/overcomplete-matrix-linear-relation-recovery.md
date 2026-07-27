---
type: technique
tags: [crypto, matrix, lattice, linear-relations, lll, technique]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/D3CTF2024-d3matrix1-wp.md
  - ../raw/crypto/WMCTF2024-matrix3-wp.md
updated: 2026-07-27
---

# Overcomplete Matrix Linear-Relation Recovery

## 适用场景

公开矩阵数量超过矩阵空间维数，秘密矩阵元素又来自很小集合。即使所有矩阵经过同一个未知共轭或线性变换，过完备集合仍会泄露模线性关系；这些关系的正交补中可能保留秘密的小元素结构或等价密钥。

## 识别信号

- 有 `k` 个 `n x n` 矩阵且 `k > n^2`，或展平向量明显线性相关。
- 私有矩阵元素来自 `{0,1}`、小整数或窄区间。
- 公开操作对所有矩阵施加相同共轭/基变换。
- 验证器接受任意等价结构，而不要求恢复原始排列或唯一私钥。

## 最小证据

- 明确运算域和模数，按同一顺序把每个矩阵展平。
- 计算公开矩阵关系空间的维数，确认存在超过偶然碰撞的线性依赖。
- 验证关系在未知共轭下保持成立。
- 确认最终目标需要原始密钥、等价密钥、矩阵和还是某个不变量。

## 解法骨架

1. 将每个公开矩阵展平成长度 `n^2` 的列或行，构造模 `p` 线性系统。
2. 求关系核，得到系数向量格；必要时通过嵌入把模关系转成整数格。
3. 求关系格的正交格，使原始小元素矩阵行/列落入其中。
4. 对小元素做中心化，并用 LLL/BKZ 恢复短向量结构。
5. 根据验证目标消除符号、平移和行置换；若等价密钥已足够，不强求唯一原像。

## 关键变体

- `{0,1}` 数据先中心化到对称区间，能显著改善短向量可见性。
- 常数平移可通过追加坐标或一行常数纳入格关系。
- 恢复结果可能只确定到行置换、符号或共轭等价类。
- 维数刚刚过完备时关系少，可能需要更多公开样本或更强 BKZ。

## 常见陷阱

- 只因 `k > n^2` 就认定能恢复秘密，没有检查小元素先验是否进入短向量。
- 展平顺序前后不一致，导致关系核和重构矩阵错位。
- 在模域求出的关系直接按普通整数使用，漏掉模数嵌入。
- 验证器只需等价密钥，却花费大量时间消除无关的排列歧义。

## 关联技巧

- [lattice-and-lwe.md](lattice-and-lwe.md)
- [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md)
- [ordered-matrix-product-trace-peeling.md](ordered-matrix-product-trace-peeling.md)
- [crypto-tooling.md](crypto-tooling.md)

## 原始资料

- [D3CTF2024-d3matrix1-wp.md](../raw/crypto/D3CTF2024-d3matrix1-wp.md)
- [WMCTF2024-matrix3-wp.md](../raw/crypto/WMCTF2024-matrix3-wp.md)
