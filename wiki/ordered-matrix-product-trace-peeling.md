---
type: technique
tags: [crypto, matrix, trace, invariant, ordered-product, technique]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/D3CTF2024-d3matrix2-wp.md
updated: 2026-07-27
---

# Ordered Matrix Product Trace Peeling

## 适用场景

秘密是公开矩阵的某个有序乘积，方案再用共轭或相似变换隐藏具体基底。目标不是直接求离散对数，而是利用迹、行列式或特征多项式等共轭不变量识别乘积两端并逐轮恢复顺序。

## 识别信号

- 公布一组小元素矩阵和一个被相似变换隐藏的总乘积。
- 秘密由矩阵排列、路径或左右乘顺序编码。
- 直接比较矩阵元素无效，但 `trace(P)`、`det(P)` 或特征多项式不受共轭影响。
- 实例矩阵具有非负、小元素或其它统计结构，使乘法后的不变量呈可区分趋势。

## 最小证据

- 验证目标矩阵与真实乘积之间确为共轭/相似关系。
- 在题目给定矩阵上实测不变量随左右乘、剥离的分布，不把“迹增大”当作一般定理。
- 确认矩阵可逆性以及左剥离、右剥离所需的模逆或精确算术。
- 用小规模排列穷举验证判定规则能区分正确端点。

## 解法骨架

1. 计算公开矩阵和目标矩阵的迹、行列式、特征多项式等不变量。
2. 对每个候选因子分别测试左剥离和右剥离后的不变量变化。
3. 根据实例上已验证的统计规律筛出可能的首尾因子。
4. 两侧同时有候选时保留分支，比较双边剥离顺序，不按首次命中贪心定死。
5. 递归到短序列后完整复乘，并通过共轭不变量或原验证器确认顺序。

## 关键变体

- 某些实例只有迹有区分度，另一些需要组合迹、行列式和多个幂次的迹。
- 矩阵不可逆时不能直接剥离，需要改用前缀/后缀搜索或其它半群不变量。
- 多个候选统计值接近时，应保留 beam search，而不是用浮点排序硬选一个。
- 双边剥离的相对顺序可能影响中间矩阵，需明确左、右乘约定。

## 常见陷阱

- 将非负矩阵上观察到的迹趋势误写成任意矩阵的数学定理。
- 混淆 `A1*A2*...` 与代码中列表、行向量/列向量的顺序。
- 用浮点矩阵求逆，舍入误差让不变量比较失真。
- 找到一条局部匹配路径后未重算完整乘积。

## 关联技巧

- [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md)
- [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md)
- [overcomplete-matrix-linear-relation-recovery.md](overcomplete-matrix-linear-relation-recovery.md)
- [crypto-tooling.md](crypto-tooling.md)

## 原始资料

- [D3CTF2024-d3matrix2-wp.md](../raw/crypto/D3CTF2024-d3matrix2-wp.md)
