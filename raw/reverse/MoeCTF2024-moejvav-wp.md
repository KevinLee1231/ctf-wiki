# moejvav

## 题目简述

Java 程序实现了一台逐字节虚拟机。每个输入字节先统一执行 `xor 0xca` 和 `add 0x20`，随后 44 个字节被分成四组，分别经过不同的 XOR/ADD 指令并与有符号 Java `byte` 常量比较。

## 解题过程

恢复出的 opcode 为：

| opcode | 语义 |
| --- | --- |
| `0` | 读取下一个输入字节到 `store` |
| `1` | `store ^= operand` |
| `2` | `store += operand` |
| `3` | `store -= operand` |
| `4` | 左移 |
| `5` | 按位或 |
| `6` | 与 operand 比较 |
| `7` | 退出 |

题目中的负目标值只是 Java 有符号 `byte` 的十进制显示；用 8 位 Z3 `BitVec` 建模即可自动按模 256 处理。四组校验分别是：

```python
targets = [
    [-25, -27, -33, -31, -50, -36, -39, -24, -52, -29, -52],
    [-64, -58, -63, -52, -90, -39, -43, 26, 25, -49, -64],
    [-51, 25, -45, -55, -47, 24, -41, -60, 22, -40, -60],
    [-15, 50, -51, -31, 50, 50, -35, 50, -35, 51, -17],
]
```

完整求解脚本：

```python
from z3 import BitVec, Solver, sat

targets = [
    [-25, -27, -33, -31, -50, -36, -39, -24, -52, -29, -52],
    [-64, -58, -63, -52, -90, -39, -43, 26, 25, -49, -64],
    [-51, 25, -45, -55, -47, 24, -41, -60, 22, -40, -60],
    [-15, 50, -51, -31, 50, 50, -35, 50, -35, 51, -17],
]

plain = [BitVec(f"ch_{i}", 8) for i in range(44)]
stage = [(char ^ 0xCA) + 0x20 for char in plain]
solver = Solver()

for i, target in enumerate(targets[0]):
    solver.add((stage[i] ^ 60) - 20 == target)
for i, target in enumerate(targets[1], start=11):
    solver.add((stage[i] ^ 14) + 5 == target)
for i, target in enumerate(targets[2], start=22):
    solver.add((stage[i] ^ 10) + 5 == target)
for i, target in enumerate(targets[3], start=33):
    solver.add(stage[i] + 14 + 10 == target)

assert solver.check() == sat
model = solver.model()
answer = bytes(model.eval(char).as_long() for char in plain)
print(answer.decode())
```

输出：

```text
moectf{jvav_eXcEpt10n_h4ndl3r_1s_s0_c00o0o1}
```

## 方法总结

这类 VM 不必完整反编译成高级语言；先做 opcode 表，再把每条线性、按位变换翻译为同宽位向量约束即可。Java `byte` 是有符号显示、8 位存储，负常量不应被当作无界负整数处理。
