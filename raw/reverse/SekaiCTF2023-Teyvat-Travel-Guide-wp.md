# Teyvat Travel Guide

## 题目简述

程序生成一张 $48\times48$ 的地图，玩家从左上角出发，只能向下或向右移动，初始生命值为 333。进入格子后会加上该格权值；过程中生命值不能小于等于 0，到达右下角时还必须恰好等于 1，才能输出 flag。

地图没有显示权值，但生成过程使用固定随机种子 `0x727`。因此关键是从二进制恢复或重建整张权值表，再用动态规划寻找满足条件的路径，而不是盲猜 94 次移动。

## 解题过程

逆向 Go 程序可还原地图生成逻辑：

```go
const LENGTH = 48

rand.Seed(0x727)
values := make([][]int, LENGTH)
for i := 0; i < LENGTH; i++ {
    values[i] = make([]int, LENGTH)
    for j := 0; j < LENGTH; j++ {
        values[i][j] = rand.Intn(51) - 40
    }
}
for i := 0; i < LENGTH; i++ {
    rand.Shuffle(LENGTH, func(j, k int) {
        values[i][j], values[i][k] =
            values[i][k], values[i][j]
    })
}
values[0][0] = 0
```

每格权值位于 $[-40,10]$。可以使用与题目相同版本的 Go 运行时重放随机序列，也可以在调试器中停在逐行洗牌完成之后，直接导出连续的 48 行切片。后者不受不同 Go 版本随机数实现差异的影响。

地图固定后，令 $dp[i][j]$ 表示走到 $(i,j)$ 时能够保留的最大生命值。到达同一格后，未来可选路径只与坐标和当前生命值有关，因此生命值较小的状态不会优于较大的状态，可以只保留最大值：

$$
dp[i][j] = v[i][j] + \max\left(dp[i-1][j],dp[i][j-1]\right).
$$

同时记录最大值来自上方还是左侧：

```python
n = 48
dp = [[-10**9] * n for _ in range(n)]
prev = [[""] * n for _ in range(n)]
dp[0][0] = 333

for i in range(n):
    for j in range(n):
        if i == 0 and j == 0:
            continue

        if i > 0 and dp[i - 1][j] > 0:
            candidate = dp[i - 1][j] + values[i][j]
            if candidate > dp[i][j]:
                dp[i][j] = candidate
                prev[i][j] = "U"

        if j > 0 and dp[i][j - 1] > 0:
            candidate = dp[i][j - 1] + values[i][j]
            if candidate > dp[i][j]:
                dp[i][j] = candidate
                prev[i][j] = "L"

assert dp[-1][-1] == 1
```

从 `(47,47)` 按 `prev` 反向回溯到 `(0,0)`，把来自上方记为正向移动 `D`、来自左侧记为 `R`，再将路径反转。对恢复出的 94 步路径重新模拟，确认每个前缀的生命值都大于 0，最终生命值为 1。

连接服务后先完成 proof of work，再逐行发送路径：

```python
for direction in path:
    io.sendline(direction.encode())
```

程序抵达右下角后输出：

```text
SEKAI{Klee_was_a_brave_girl_today!_I_found_a_really_weird-looking_lizard!_Want_me_to_show_it_to_you?}
```

## 方法总结

固定种子的伪随机数据不具备保密性；若运行时重放可能受版本影响，直接从内存导出最终矩阵更稳妥。本题的路径状态满足最优子结构：坐标相同且生命值更高的状态支配生命值更低的状态，所以二维最大值 DP 足够，无需枚举 $\binom{94}{47}$ 条路径。回溯后仍应完整模拟一次，以核验“途中始终存活”和“终点恰为 1”两个条件。
