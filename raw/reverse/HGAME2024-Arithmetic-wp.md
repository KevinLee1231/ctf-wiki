# Arithmetic

## 题目简述

附件是修改了 UPX 特征的可执行文件，并会从 `out` 读取一个 500 层数字三角形。程序每层向左下或右下选择一个数，累加路径和，并对符合条件的路径序列求 MD5。逆向得到规则后，问题可转化为经典“数字金字塔”最大路径动态规划。

## 解题过程

### 修复 UPX 特征并脱壳

程序使用 UPX 加壳，但头部的 `UPX` 特征字节被改成了 `ari`，导致常规脱壳命令无法识别。在十六进制编辑器中定位被破坏的 UPX 标记，将 `ari` 恢复为 `UPX`，再执行：

```bash
upx -d arithmetic
```

脱壳后载入 IDA，可看到程序从 `out` 中逐个读取整数，按第 $1,2,3,\ldots,500$ 层保存为三角形数组。

### 恢复路径规则

每步会取值 `1` 或 `2`：

- `1` 表示走到正下方，下一层的列索不变；
- `2` 表示走到右下方，下一层的列索加一。

程序将从顶层到底层的所有数求和，并以 `6752833` 作为校验目标。官方题解指出有效路径唯一，而该路径就是三角形的最大和路径。因此不需要枚举 $2^{499}$ 种选择，用动态规划记录每个位置的最大前缀和与前驱即可。

转移方程为：

$$
dp_{i,j}=a_{i,j}+\max(dp_{i-1,j},dp_{i-1,j-1}).
$$

其中从 $(i-1,j)$ 转移到 $(i,j)$ 对应动作 `1`，从 $(i-1,j-1)$ 转移对应动作 `2`。

### 重建路径并计算 MD5

以下脚本从 `out` 的扁平数据恢复三角形，计算最大和，回溯得到由 `1`/`2` 组成的 499 步路径，最后对路径字符串求 MD5：

```python
from hashlib import md5
from pathlib import Path

numbers = [int(x) for x in Path("out").read_text().split()]

rows = []
offset = 0
width = 1
while offset < len(numbers):
    row = numbers[offset:offset + width]
    if len(row) != width:
        raise ValueError("out 不是完整的三角形数据")
    rows.append(row)
    offset += width
    width += 1

dp = [rows[0][0]]
parents = []

for i in range(1, len(rows)):
    next_dp = []
    row_parents = []
    for j, value in enumerate(rows[i]):
        from_same = dp[j] if j < len(dp) else None
        from_left = dp[j - 1] if j > 0 else None

        if from_left is None or (
            from_same is not None and from_same >= from_left
        ):
            next_dp.append(from_same + value)
            row_parents.append((j, "1"))
        else:
            next_dp.append(from_left + value)
            row_parents.append((j - 1, "2"))

    dp = next_dp
    parents.append(row_parents)

column = max(range(len(dp)), key=dp.__getitem__)
maximum = dp[column]

moves = []
for i in range(len(rows) - 1, 0, -1):
    column, move = parents[i - 1][column]
    moves.append(move)
moves.reverse()

path = "".join(moves)
print("rows:", len(rows))
print("maximum:", maximum)
print("path length:", len(path))
print("md5:", md5(path.encode()).hexdigest())
```

在题目数据上，最大路径和为 `6752833`，路径 MD5 为：

```text
934f7f68145038b3b81482b3d9f3a355
```

因此 flag 是：

```text
hgame{934f7f68145038b3b81482b3d9f3a355}
```

## 方法总结

- 壳的特征被少量修改时，可结合节名、入口 stub 和字节特征判断原壳，修复最小必要标记后再用原工具脱壳。
- 逆向的结果不一定是某个密码算法；本题最终是将程序行为转写为标准动态规划模型。
- 除了最优值，还要保留前驱才能重建路径；本题的 flag 依赖路径本身的 MD5，只算出和还不够。
- 如果最大值存在并列路径，应枚举所有并列前驱并逐一校验哈希；“路径唯一”是本题数据的性质，不是动态规划的普遍保证。
