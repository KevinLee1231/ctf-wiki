# N1CTF 2022 - flag_compiler

## 题目简述

题目把 flag 校验器实现成 C++ 模板元编程：模板类型充当函数，模板参数保存数据，整个检查在编译期执行。展开 emoji 宏和二进制类型常量后，可以识别出一台栈式虚拟机，状态由指令指针、栈指针和 17 个 32 位字组成，指令保存在模板类型构成的 ROM 中。

仓库只给出简要说明和正向参考脚本。NeSE Team 的 [N1CTF 2022 题解 PDF](https://nese.team/writeup/n1ctf2022.pdf) 保存了从模板展开、VM 反汇编到逆运算脚本的完整过程；相关页面已逐页核对，代码截图均为纯文本信息，下面将其转写并整理为可复现解法。

## 解题过程

### 展开模板并识别 VM

先给重复出现的 emoji 标识符重命名，再把形如 `_BinNumber<struct_1, struct_2, ...>` 的小端二进制模板还原为整数。基础模板分别实现：

- 类型相等和数值相等；
- `Inc`、`Dec`；
- `uint_list`、`type_list` 的取首项、弹出、索引、切片和赋值；
- `state<IP, SP, RAM>`；
- 根据 ROM 当前项递归实例化下一状态的 `Eval`。

程序以 `IP >= 2 << 10` 作为停止条件，最终检查 RAM 第 0 项是否为 1。还原出的主要 opcode 如下：

| opcode | 语义 |
| --- | --- |
| `op00` | `pop` |
| `op01` | 压入立即数 |
| `op02` | `dup` |
| `op03` | 绝对跳转 |
| `op04` | 比较并条件跳转 |
| `op05` / `op06` | 间接 `load` / `store` |
| `op07` / `op08` | 从固定 RAM 下标压栈 / 弹栈到固定下标 |
| `op09` | 加法 |
| `op10` / `op11` / `op12` | 立即数乘法、乘法和无符号乘法 |
| `op13` | 自增 |
| `op14` | 不等时相对跳转 |
| `op15` | 清空栈 |
| `op16` | 右移一位并按最低位跳转 |

将模板 ROM 抄成普通指令后，可以看出 `op16` 与乘法指令共同实现二进制快速幂，其余循环实现两轮线性变换。

### 还原正向检查

输入补零到 40 字节，再按小端序拆成 10 个 `uint32`。设状态为 $R_0,\ldots,R_9$，滚动乘子初值和模数为：

$$
K=133723339=\mathtt{0x7f874cb},\qquad
M=4200000037=\mathtt{0xfa56ea25}.
$$

第一轮对每个字执行仿射变换：

$$
R_j\leftarrow(KR_j+j)\bmod M,
$$

每处理一个 $j$，还要令 $K\leftarrow133723339K\bmod M$。

第二轮重复 35 次。每轮先保存旧的 $R_0,R_1$ 作为循环尾部的相邻项，然后依次计算：

$$
R_i\leftarrow K(R_i+2R_{i+1}+3R_{i+2})\bmod M,
$$

下标按模 10 循环，且每个 $i$ 后同样更新 $K$。35 轮结束时：

```text
K = 1849312651 = 0x6e3a458b
```

最后逐项计算：

$$
T_i=4294967279^{R_i}\bmod4294967291.
$$

VM 中保存的目标数组为：

```python
target = [
    1520964462, 1435920992,   27625034, 2434818676, 2801894050,
     375115101, 4005739747, 3182810789, 2371651904, 4204614650,
]
```

仓库的 `ref.py` 正向运行后得到的正是该数组，说明上述反编译结果与官方参考实现一致。

### 逆向快速幂和两轮线性变换

先在模 $P=4294967291$ 下求离散对数，恢复快速幂之前的 10 个指数：

```python
state = [
    int(discrete_log(0xfffffffb, value, 0xffffffef))
    for value in target
]
```

第二轮每次都是模 $M$ 上的十元线性方程组。系数矩阵第 $i$ 行仅在 $i$、$i+1$、$i+2$ 三列上分别为 1、2、3。逆序回退滚动乘子，将当前结果除以该行使用的 $K$，再做模高斯消元，即可恢复上一轮状态。重复 35 次后，再逆序撤销第一轮的仿射变换。

完整求解脚本如下：

```python
from sympy.ntheory import discrete_log

M = 0xFA56EA25
G = 0x07F874CB
P = 0xFFFFFFFB
BASE = 0xFFFFFFEF

target = [
    1520964462, 1435920992,   27625034, 2434818676, 2801894050,
     375115101, 4005739747, 3182810789, 2371651904, 4204614650,
]

def solve_linear(matrix, vector, mod):
    n = len(matrix)
    aug = [
        [matrix[i][j] % mod for j in range(n)] + [vector[i] % mod]
        for i in range(n)
    ]

    for col in range(n):
        pivot = next(row for row in range(col, n) if aug[row][col])
        aug[col], aug[pivot] = aug[pivot], aug[col]

        inv = pow(aug[col][col], -1, mod)
        aug[col] = [(value * inv) % mod for value in aug[col]]

        for row in range(n):
            if row == col:
                continue
            factor = aug[row][col]
            if factor:
                aug[row] = [
                    (aug[row][j] - factor * aug[col][j]) % mod
                    for j in range(n + 1)
                ]

    return [aug[i][-1] for i in range(n)]

state = [int(discrete_log(P, value, BASE)) for value in target]
k = 0x6E3A458B
g_inv = pow(G, -1, M)

while k != pow(G, 11, M):
    matrix = []
    for i in range(10):
        row = [0] * 10
        row[i] = 1
        row[(i + 1) % 10] = 2
        row[(i + 2) % 10] = 3
        matrix.append(row)

    vector = [0] * 10
    for i in range(9, -1, -1):
        k = k * g_inv % M
        vector[i] = state[i] * pow(k, -1, M) % M

    state = solve_linear(matrix, vector, M)

for j in range(9, -1, -1):
    k = k * g_inv % M
    state[j] = (state[j] - j) * pow(k, -1, M) % M

assert k == G
flag = b"".join(value.to_bytes(4, "little") for value in state)
print(flag.rstrip(b"\x00"))
```

本地执行得到：

```text
C++:teMp1aTe<Vm>_1s_cOo0l-6ut~Slow!;#<=>
```

再把该字节串交给仓库的正向 `ref.py`，输出与 VM 中的十个目标值完全一致。

## 方法总结

模板元编程混淆不能只看类型名，应先把容器、状态和递归求值器映射成普通数据结构，再恢复 opcode。VM 的最终算法由“十个小端 `uint32`、一轮仿射变换、35 轮循环线性变换、模幂比较”组成。逆向时依次求离散对数、逆线性方程组、逆仿射变换，并在最后用官方正向脚本验证，能够把每一层都闭环确认。
