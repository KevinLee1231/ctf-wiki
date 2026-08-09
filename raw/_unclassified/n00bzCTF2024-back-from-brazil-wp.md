# Back From Brazil

## 题目简述

每轮给出 1000×1000 的权值网格，只能向右或向下，从左上走到右下并取得最大路径和；共需在 30 秒内完成 10 轮。穷举路径不可行，必须使用二维动态规划并回溯路径。

## 解题过程

定义 `dp[i][j]` 为到达 $(i,j)$ 的最大权值和：

$$dp[i][j]=eggs[i][j]+\max(dp[i-1][j],dp[i][j-1]).$$

同时记录最优前驱方向：

```python
for i in range(n):
    for j in range(n):
        if i == 0 and j == 0:
            dp[i][j] = eggs[i][j]
            continue
        up = dp[i - 1][j] if i else -1
        left = dp[i][j - 1] if j else -1
        if up >= left:
            dp[i][j] = up + eggs[i][j]
            prev[i][j] = "r"
        else:
            dp[i][j] = left + eggs[i][j]
            prev[i][j] = "d"
```

从右下角逆向回溯并反转，得到长度恰为 $2n-2$ 的 `r`/`d` 路径。连续提交 10 轮后，实际源码和官方 solver 对应的 flag 是：

```text
n00bz{1_g0t_b4ck_h0m3!!!}
```

仓库 README 写成了另一句，但与完整服务端源码、`flag.txt` 和求解脚本不一致，故不采用。

## 方法总结

网格单调路径具有重叠子问题，二维 DP 将指数复杂度降为 $O(n^2)$。只算最大和不够，服务还验证完整路径长度，因此必须保存前驱并回溯。
