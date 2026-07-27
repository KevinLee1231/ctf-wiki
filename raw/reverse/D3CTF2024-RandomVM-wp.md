# RandomVM

## 题目简述

程序把校验逻辑编译成一种类似 Brainfuck 的简易虚拟机，并用 `rand()` 选择分散的基本块，试图增加直接 dump 或 patch 字节码的难度。部分 VM 指令还调用 `ptrace(PTRACE_TRACEME)`：未被调试时第一次调用返回 0，后续调用返回 -1；已经处于调试器中时，第一次调用也会失败，随机分支因而发生变化。

这套保护需要控制流能展开为树状结构，适合固定的加密或哈希流程，但通用性有限。题目实现中，反调试差异实际只影响开头少数字节；即使不处理 `ptrace`，也能通过静态恢复、非 `ptrace` 跟踪或小规模枚举得到有效校验逻辑。

## 解题过程

### 恢复 VM 指令语义

查询 `rand()` 的交叉引用，可以定位虚拟机相关基本块。各符号语义如下：

| 指令 | 语义 |
|---|---|
| `>` / `<` | `flagptr` 加一 / 减一 |
| `]` / `[` | `slotptr` 加一 / 减一 |
| `0` | 当前 `slot` 清零 |
| `+` / `-` | 当前 `slot` 加一 / 减一 |
| `,` / `.` | 从 `flag` 读入当前 `slot` / 写回 `flag` |
| `^` | 当前 flag 字节与当前 `slot` 异或 |
| `x` | 按当前 `slot` 指定的位数循环右移 flag 字节 |
| `s` | 用连续 slot 作为参数发起系统调用 |
| `T` | 当前 slot 为负时调用 `rand()`，参与分支扰动 |

静态方案可以用 IDAPython 从 `rand()` 的交叉引用出发恢复块间关系；动态方案可用 uprobe 等不依赖 `ptrace` 的跟踪机制记录实际经过的块。恢复后的大量 `0-T` 片段主要用于扰动，真正的逐字节变换可压缩为两个参数表：

```python
rotations = [3, 5, 6, 7, 4, 4, 7, 7, 2, 4, 4, 7]
use_xor =   [1, 0, 0, 1, 1, 0, 1, 0, 0, 0, 0, 1]
```

目标密文为：

```python
cipher = [
    0x9D, 0x6B, 0xA1, 0x02, 0xD7, 0xED,
    0x40, 0xF6, 0x0E, 0xAE, 0x84, 0x19,
]
```

### 写出等价数学变换

设原始输入为 $a_0,\ldots,a_{11}$，循环右移位数为 $k_i$。第一阶段先计算：

$$
r_i=\operatorname{ROR}_8(a_i,k_i)\oplus
\begin{cases}
k_i,&\text{该位置存在异或}\\
0,&\text{否则}
\end{cases}
$$

除最后一位外，还会与下一原始字节异或：

$$
b_i=r_i\oplus a_{i+1}\quad(0\le i<11),\qquad b_{11}=r_{11}
$$

第二阶段执行前缀异或：

$$
c_0=b_0,\qquad c_i=b_i\oplus c_{i-1}
$$

最终 $c_i$ 与题目给出的 `cipher[i]` 比较。原题解使用 Z3 建模即可求解，但这组运算全部可逆，无需 SMT 也能直接从后向前还原。

### 直接逆变换

先消去前缀异或：

$$
b_0=c_0,\qquad b_i=c_i\oplus c_{i-1}
$$

再从 $a_{11}$ 向 $a_0$ 逆推。循环右移的逆运算是循环左移，因此：

$$
a_{11}=\operatorname{ROL}_8(b_{11}\oplus m_{11},k_{11})
$$

$$
a_i=\operatorname{ROL}_8(b_i\oplus a_{i+1}\oplus m_i,k_i)
$$

其中 $m_i$ 在 `use_xor[i] == 1` 时等于 $k_i$，否则为 0。

完整求解脚本如下：

```python
#!/usr/bin/env python3

cipher = [
    0x9D, 0x6B, 0xA1, 0x02, 0xD7, 0xED,
    0x40, 0xF6, 0x0E, 0xAE, 0x84, 0x19,
]
rotations = [3, 5, 6, 7, 4, 4, 7, 7, 2, 4, 4, 7]
use_xor = [1, 0, 0, 1, 1, 0, 1, 0, 0, 0, 0, 1]


def rol8(value: int, count: int) -> int:
    count %= 8
    return ((value << count) | (value >> (8 - count))) & 0xFF


intermediate = [cipher[0]]
intermediate.extend(
    cipher[index] ^ cipher[index - 1]
    for index in range(1, len(cipher))
)

plain = [0] * len(cipher)
last = len(cipher) - 1
last_mask = rotations[last] if use_xor[last] else 0
plain[last] = rol8(
    intermediate[last] ^ last_mask,
    rotations[last],
)

for index in range(last - 1, -1, -1):
    mask = rotations[index] if use_xor[index] else 0
    value = intermediate[index] ^ plain[index + 1] ^ mask
    plain[index] = rol8(value, rotations[index])

body = bytes(plain).decode()
print(body)
print(f"d3ctf{{{body}}}")
```

输出为：

```text
m3owJumpVmvM
d3ctf{m3owJumpVmvM}
```

## 方法总结

逆向重点是把随机化的基本块调度与真实 VM 数据流分开。`rand()` 交叉引用、系统调用块和会修改 `flagptr/slotptr` 的块是恢复控制流的有效锚点；重复的 `0-T` 只影响分支选择，不应原样堆进最终 WP。

恢复等价变换后，应先判断运算是否可逆。本题只有 8 位循环移位和异或，直接逆推比调用 Z3 更简单，也能清楚说明为什么结果唯一。反调试实现未覆盖全部字节，说明保护设计还需验证不同运行状态下的实际影响，不能只根据设计意图判断强度。
