# Nokotan's Antlers Won't Stop Growing, So Let's Play a Game on a Binary Tree!

## 题目简述

给定按堆顺序组成的 $n$ 节点完全二叉树。玩家可以给所有叶子任意标记 $0$ 或 $1$；随后自底向上令每个内部节点的标记等于子节点标记的异或，缺失子节点等价于 $0$。树的 value 是标记为 $1$ 的节点总数，要求统计所有叶子初始标记能够产生多少种不同 value。

约束为多组数据总 $n\le5\times10^5$。题面 PDF 的动画截图只用于剧情，约束和状态转移均可用文本完整表示，因此不保留图片。

## 解题过程

### 用生成函数表示子树状态

对每种子树维护两个布尔多项式：

- $F_0(x)$：根标记为 $0$ 时，可实现的子树 value。
- $F_1(x)$：根标记为 $1$ 时，可实现的子树 value。

若第 $k$ 项系数非零，表示 value $k$ 可达。叶子的初始状态为：

$$
F_0(x)=1,\qquad F_1(x)=x.
$$

设左右子树分别为 $(L_0,L_1)$、$(R_0,R_1)$。异或结果为 $0$ 时两边根标记相同；结果为 $1$ 时两边不同，且当前根自身还贡献一个 $1$：

$$
F_0=\operatorname{bool}(L_0R_0+L_1R_1),
$$

$$
F_1=x\cdot\operatorname{bool}(L_0R_1+L_1R_0).
$$

多项式乘法的卷积正好枚举左右子树 value 之和。每次卷积后只关心系数是否为零。

### 利用完全二叉树只有少量不同形状

若对每个节点独立维护多项式，即使使用 FFT，总开销仍然过大。完全二叉树具有更强结构：

- 同一高度的满子树完全相同，只需计算一次。
- 每一高度至多有一个包含最后一层缺口的非满子树。

官方代码先为每个节点计算 `(是否为满子树, 高度)`。然后分别缓存：

```cpp
dpFull0[height], dpFull1[height]
dpPartial0[height], dpPartial1[height]
```

满子树的转移只由前一高度的同一组状态产生：

```cpp
t00 = multiply(dpFull0[h - 1], dpFull0[h - 1]);
t11 = multiply(dpFull1[h - 1], dpFull1[h - 1]);
t01 = multiply(dpFull0[h - 1], dpFull1[h - 1]);
t10 = multiply(dpFull1[h - 1], dpFull0[h - 1]);

dpFull0[h] = boolean_union(t00, t11);
dpFull1[h] = shift_one(boolean_union(t01, t10));
```

非满子树根据缺口在左侧还是右侧，把一个 `Partial` 状态与对应高度的 `Full` 状态卷积。只有一个左孩子的两节点子树是特殊基例：根为 $0$ 时 value 为 $0$，根为 $1$ 时根和孩子都为 $1$，value 为 $2$。

卷积使用 FFT，所有高度上的多项式总长度形成几何级数，因此总体复杂度约为 $O(n\log n)$，空间复杂度为 $O(n)$。

### 统计根节点答案

在根的 $F_0,F_1$ 中收集所有非零系数下标并去重：

```cpp
set<int> values;
for (int k = 0; k < root0.size(); ++k)
    if (root0[k]) values.insert(k);
for (int k = 0; k < root1.size(); ++k)
    if (root1[k]) values.insert(k);
cout << values.size() << '\n';
```

## 方法总结

- 核心技巧：用布尔生成函数编码可达 value，异或状态转移化为四次卷积；再利用完全二叉树每层至多两种子树形状进行复用。
- 识别信号：树 DP 的状态是“可达和集合”，左右子树组合需要 Minkowski sum；同时输入树具有完全、满或周期性结构。
- 复用要点：仅把 $O(s^2)$ 合并换成 FFT 还不一定足够，还要统计相同子问题的重复次数。卷积系数只表示可达性时，应在每次运算后布尔化，避免无意义的大整数增长。
