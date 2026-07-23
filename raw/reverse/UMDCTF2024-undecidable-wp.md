# undecidable

## 题目简述

附件是一个由数千个小函数组成的二进制。每个函数根据纸带当前位选择“写位、左移或右移、跳转下一状态”，本质上是一台状态编号被随机打乱的图灵机。输入长度提示为 46 字节，即 368 位；目标是从状态图中分离真正的线性校验器，逆解正确纸带内容。

## 解题过程

生成源码表明纸带用 `pos`、`neg` 两个动态位数组保存。命令行输入按最低位优先写入偶数位置：

```c
flip(2 * (byte_index * 8 + bit_index));
```

位置 $-1$ 和 `strlen(input) * 16 - 1` 还会写入边界标记。每个 `fp_*` 函数都能还原为一条标准转移：

```text
state  write_if_0  move_if_0  next_if_0  write_if_1  move_if_1  next_if_1
```

虽然函数名和 `fps[]` 下标经过随机置换，但每个分支中的 `flip(s)`、`s += 1`/`s -= 1` 和下一函数指针都是显式的，因此可以从入口状态沿控制流恢复逻辑状态图。

状态图前半段先删除输入位之间的空格并重排纸带。后半段对 368 位执行两轮逐渐收缩的相邻异或：

```python
start = 0
end = 367
for _ in range(184):
    for j in range(start, end):
        bits[j] ^= bits[j + 1]
    end -= 1
    for j in range(end, start, -1):
        bits[j] ^= bits[j - 1]
    start += 1
```

最后 368 个判断状态逐位比较变换结果：正确位进入下一状态，错误位跳进附加的干扰图灵机。于是整个有效校验可写成有限域 $\operatorname{GF}(2)$ 上的线性系统：

$$
M x=t
$$

其中 $x$ 是输入的 368 个低位优先比特，$M$ 可通过对单位矩阵执行同样的相邻异或得到，$t$ 则由每个比较状态中“哪一条分支继续前进”恢复。

公开仓库中的 `checker.txt` 是从二进制状态函数还原后的未打乱转移表。下面的纯 Python 脚本从末尾状态提取 $t$，构造 $M$ 并用异或高斯消元求解：

```python
from pathlib import Path

n = 46 * 8
transitions = {}
for line in Path("checker.txt").read_text().splitlines():
    state, w0, d0, n0, w1, d1, n1 = line.split()
    transitions[int(state)] = (int(n0), int(n1))

max_state = max(transitions)
sink = max_state + 1
base = max_state - 2 * n

target = []
for i in range(n):
    state = base + 2 * i
    next_zero, next_one = transitions[state]
    assert {next_zero, next_one} == {state + 1, sink}
    target.append(0 if next_zero == state + 1 else 1)

matrix = [[int(i == j) for j in range(n)] for i in range(n)]
start, end = 0, n - 1
for _ in range(n // 2):
    for j in range(start, end):
        matrix[j] = [a ^ b for a, b in zip(matrix[j], matrix[j + 1])]
    end -= 1
    for j in range(end, start, -1):
        matrix[j] = [a ^ b for a, b in zip(matrix[j], matrix[j - 1])]
    start += 1

augmented = [matrix[i] + [target[i]] for i in range(n)]
row = 0
pivots = []
for column in range(n):
    pivot = next(r for r in range(row, n) if augmented[r][column])
    augmented[row], augmented[pivot] = augmented[pivot], augmented[row]
    for r in range(n):
        if r != row and augmented[r][column]:
            augmented[r] = [
                a ^ b for a, b in zip(augmented[r], augmented[row])
            ]
    pivots.append(column)
    row += 1

solution = [0] * n
for r, column in enumerate(pivots):
    solution[column] = augmented[r][-1]

flag = bytes(
    sum(solution[i + bit] << bit for bit in range(8))
    for i in range(0, n, 8)
)
print(flag.decode())
```

输出：

```text
UMDCTF{can_a_mentat_prove_its_own_consistency}
```

## 方法总结

题目的“不可判定”外观来自图灵机和错误分支中的干扰状态，但固定长度输入只经过有限、可恢复的状态图。识别出相邻异或后，校验器就是 $\operatorname{GF}(2)$ 上的满秩线性变换。最稳妥的做法是让单位矩阵经历同一变换来自动生成 $M$，而不是手推 368 个输出位的公式。
