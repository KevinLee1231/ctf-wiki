# Pafs

## 题目简述

给定一棵 $n$ 个点的树。一个 Paf 是非空边序列，序列中的边两两不同，且任意两条相邻边共享至少一个端点；边的顺序不同视为不同 Paf。任务是统计所有 Paf 的数量，并对 $10^9+7$ 取模。

原题 PDF 共 5 页，已逐页核对定义、约束和样例。仓库中的官方说明给出了“简单路径加局部排列”的组合解释，官方程序则把它压缩成 $O(n)$ 树形 DP。

## 解题过程

先固定 Paf 的起点和终点。树上两点间只有一条简单路径，路径边在 Paf 中必须按路径方向依次出现；但经过内部点 $v$ 时，可以在进入边和离开边之间插入若干条与 $v$ 相连、又不属于主路径的边。

若 $\deg(v)=d$，可选的额外边有 $d-2$ 条。选取其中任意 $j$ 条并排列，方案数为：

$$
P(d-2,j)=\frac{(d-2)!}{(d-2-j)!}.
$$

因此内部点的局部贡献为：

$$
\begin{aligned}
F(v)
&=\sum_{j=0}^{d-2}P(d-2,j) \\
&=\sum_{i=0}^{d-2}\frac{(d-2)!}{i!}.
\end{aligned}
$$

一条有向简单路径的贡献就是所有内部点 $F(v)$ 的乘积。直接枚举端点仍是 $O(n^2)$，需要在树上合并路径。

定义辅助函数：

$$
\operatorname{choose}(d,k)=\sum_{i=k}^{d}P(d,i),
$$

即从 $d$ 条候选边中选至少 $k$ 条并排列的方案数。其值可以在每个点处线性递推排列数：

```python
def choose(d, at_least):
    total = 1 if at_least == 0 else 0
    perm = 1
    for length in range(1, d + 1):
        perm = perm * (d + 1 - length) % MOD
        if length >= at_least:
            total = (total + perm) % MOD
    return total
```

先统计只使用一条边以及全部边都围绕同一个中心点的 Paf：

```python
answer = n - 1
for v in range(n):
    answer += choose(deg[v], 2)
```

其中 `n - 1` 是单边序列；`choose(deg[v], 2)` 是在同一星形中选择至少两条不同边并排序。

随后任选根做 DFS。对点 $v$，令 `ret` 表示一端进入 $v$、另一端落在当前已合并方向中的路径方案数：

```python
def dfs(v, parent):
    global answer

    d = len(graph[v])
    ret = choose(d - 1, 1)
    multiplier = choose(d - 2, 0)

    for child in graph[v]:
        if child == parent:
            continue

        take = dfs(child, v)
        answer = (
            answer + 2 * ret * take
        ) % MOD
        ret = (
            ret + multiplier * take
        ) % MOD

    return ret
```

`2 * ret * take` 把当前已处理方向与新子树连接，并计入正反两个边序；`multiplier` 则允许在 $v$ 处插入任意局部边排列。每条边只进入一次 DFS，各点的 `choose` 总循环次数等于度数之和，所以总复杂度为 $O(n)$。

## 方法总结

本题表面是在统计任意边序列，真正的骨架却是树上的唯一简单路径。把非路径边视为内部点的局部“部分排列”后，全局计数就能拆成路径合并。推导时先明确局部组合因子，再设计能逐子树合并有向路径的 DP 状态，比直接枚举序列更容易控制重复计数。
