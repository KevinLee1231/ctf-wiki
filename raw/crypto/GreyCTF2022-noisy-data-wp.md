# GreyCTF2022 - Noisy Data

## 题目简述

题目把 flag 位流送入码率 $1/2$ 的卷积码，生成多项式为二进制 `111` 和 `101`，再随机翻转约 $1.5\%$ 的输出位。直接逐位异或无法纠错，需要在编码器的有限状态图上做最大似然译码。

## 解题过程

约束长度为 3，因此编码器只需记住前两位，共有 4 个状态。对每个状态分别尝试输入位 0、1，计算两路理论输出与当前接收二元组的汉明距离，并累计路径代价。

```python
INF = 10**9
cost = [0, INF, INF, INF]
paths = [b"", b"", b"", b""]

for y0, y1 in pairs(received):
    new_cost = [INF] * 4
    new_paths = [b""] * 4
    for state in range(4):
        for bit in (0, 1):
            nxt, (e0, e1) = transition(state, bit)
            score = cost[state] + (e0 != y0) + (e1 != y1)
            if score < new_cost[nxt]:
                new_cost[nxt] = score
                new_paths[nxt] = paths[state] + bytes([bit])
    cost, paths = new_cost, new_paths
```

这就是 Viterbi 算法。选择总代价最小的终态路径，将比特按每 8 位还原为字节，得到：

```text
grey{l1kELy_tO_uNd3rStanD_My_m5Gs}
```

## 方法总结

卷积码的关键是“局部状态有限、全局路径很多”。Viterbi 用动态规划保留到每个状态的最佳前缀，复杂度随消息长度线性增长。实现时必须核对生成多项式的位序、初始/终止状态和字节端序，否则即使路径代价很低也可能解出乱码。
