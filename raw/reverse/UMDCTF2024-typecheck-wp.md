# Typecheck

## 题目简述

题目把校验逻辑全部写进 C++ 模板实例化。`flag_t` 是 60 个字符对应的整数类型列表，`prog_t` 则是一段模板虚拟机程序；错误输入会让 `std::enable_if_t` 实例化失败。需要解释指令集、提取线性方程并恢复能够通过编译期检查的 flag。

## 解题过程

`templates.hpp` 中五条指令的语义为：

| 操作码 | 语义 |
|---|---|
| `0` | 弹出两个值并相加 |
| `1` | 弹出两个值并相乘 |
| `2, v` | 压入常数 $v$ |
| `3, n` | 压入输入的第 $n$ 个字符 |
| `4, v` | 要求栈顶等于 $v$，相等后弹出 |

`prog_t` 反复执行如下模式：

```text
push 0
push input[index]
push coefficient
multiply
add
...
test expected
```

因此每个 `test` 都对应一条关于 60 个字符值的线性方程。整个程序恰好生成 60 条满秩方程，无需模拟数千层模板实例化，只需解析整数列表并求解矩阵。

下面的脚本直接从 `main.cpp` 提取 `prog_t`。每个栈元素用长度 61 的向量表示，前 60 项是变量系数，最后一项是常数：

```python
import re
from pathlib import Path

import numpy as np

source = Path("main.cpp").read_text()
body = re.search(
    r"using prog_t\s*=\s*int_list_t\s*<(.*?)>;",
    source,
    re.S,
).group(1)
program = list(map(int, re.findall(r"-?\d+", body)))

n = 60
stack = []
rows = []
rhs = []
pc = 0

while pc < len(program):
    opcode = program[pc]
    pc += 1

    if opcode == 0:
        a, b = stack.pop(), stack.pop()
        stack.append(a + b)
    elif opcode == 1:
        a, b = stack.pop(), stack.pop()
        a_has_var = np.any(a[:n])
        b_has_var = np.any(b[:n])
        if a_has_var and b_has_var:
            raise ValueError("unexpected nonlinear term")
        if a_has_var:
            stack.append(a * b[-1])
        elif b_has_var:
            stack.append(b * a[-1])
        else:
            value = np.zeros(n + 1, dtype=np.int64)
            value[-1] = a[-1] * b[-1]
            stack.append(value)
    elif opcode == 2:
        value = np.zeros(n + 1, dtype=np.int64)
        value[-1] = program[pc]
        pc += 1
        stack.append(value)
    elif opcode == 3:
        value = np.zeros(n + 1, dtype=np.int64)
        value[program[pc]] = 1
        pc += 1
        stack.append(value)
    elif opcode == 4:
        expected = program[pc]
        pc += 1
        expression = stack.pop()
        rows.append(expression[:n])
        rhs.append(expected - expression[-1])

matrix = np.array(rows, dtype=float)
target = np.array(rhs, dtype=float)
solution = np.rint(np.linalg.solve(matrix, target)).astype(int)

# 浮点求解后用整数方程重新验算。
assert np.all(np.array(rows, dtype=np.int64) @ solution == rhs)
print("".join(map(chr, solution)))
```

结果为：

```text
UMDCTF{c++_templates_are_the_reason_for_the_butlerian_jihad}
```

把这 60 个字符替换进 `flag_t` 后，用题目提示的深度参数编译即可验证：

```bash
g++ -std=c++17 -ftemplate-depth=10000 main.cpp -o typecheck
```

## 方法总结

模板元编程只是执行载体，底层校验仍是一组普通线性约束。先把模板偏特化翻译成小型指令集，再用符号系数向量执行 VM，就能自动生成矩阵。浮点线性求解足以给出候选，但必须回代原始整数方程，避免四舍五入掩盖错误。
