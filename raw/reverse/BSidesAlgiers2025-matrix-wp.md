# BSidesAlgiers2025 - matrix

## 题目简述

附件包含静态链接的 Go 程序 `main`、139 字节的 `program.bin` 与一个 $8\times8$ 整数矩阵 `result.txt`。直接把矩阵元素转成 ASCII 只能得到打乱文本；逆向二进制中的 `main.Matrix`、`matAdd`、`matSub` 等符号后，可判断 `program.bin` 是一个矩阵虚拟机程序，而 `result.txt` 是执行后的矩阵 0。目标是反向执行对矩阵 0 生效的行列交换。

## 解题过程

字节码指令长度固定：

- `0x30 matrix col_a col_b`：交换两列，共 4 字节；
- `0x40 matrix row_a row_b`：交换两行，共 4 字节；
- `0x50/0x60/0x70/0x80 dst src`：矩阵加、减、乘、异或，共 3 字节；
- `0xCC matrix`：打印矩阵，共 2 字节。

这份程序共有 40 条指令，其中 20 条是作用于矩阵 0 的行列交换。其余算术指令操作其他矩阵，不改变最终提供的矩阵 0；交换又是自逆操作，所以只需逆序重放 `0x30/0x40`。

完整恢复脚本如下：

```python
from pathlib import Path

root = Path(".")
matrix = [
    [int(x) for x in line.split(",")]
    for line in (root / "result.txt").read_text().splitlines()
]
program = (root / "program.bin").read_bytes()

ops = []
pc = 0
while pc < len(program):
    opcode = program[pc]
    if opcode in (0x30, 0x40):
        ops.append((opcode, program[pc + 1], program[pc + 2], program[pc + 3]))
        pc += 4
    elif opcode in (0x50, 0x60, 0x70, 0x80):
        ops.append((opcode, program[pc + 1], program[pc + 2]))
        pc += 3
    elif opcode == 0xCC:
        ops.append((opcode, program[pc + 1]))
        pc += 2
    else:
        raise ValueError(f"unknown opcode {opcode:#x} at {pc:#x}")

for op in reversed(ops):
    if op[0] == 0x30 and op[1] == 0:
        _, _, a, b = op
        for row in matrix:
            row[a], row[b] = row[b], row[a]
    elif op[0] == 0x40 and op[1] == 0:
        _, _, a, b = op
        matrix[a], matrix[b] = matrix[b], matrix[a]

print("".join(chr(value) for row in matrix for value in row))
```

本地运行得到：

`shellmates{sh3ll_m4tr1x_vM_8x8_l33t_h4ck_th3_m4th_0f_r3v_w0rld!}`

该输出也与公开的 [matrix 逆向记录](https://medium.com/@bl0rph/matrix-bsides-algiers-2k25-c1918c830a3b) 一致；外部记录用于交叉核对 opcode 语义，完整解析和求解代码已在正文给出。

## 方法总结

- 数据文件看似乱码时，应先确认它是输入、输出还是 VM 状态；本题的 `result.txt` 是终态，不能正向再跑一次。
- 自逆变换最适合逆序重放：行交换、列交换各执行一次即可撤销，不需要构造额外逆矩阵。
- 解析变长字节码时必须按 opcode 明确推进 `pc`；`0xCC` 是 2 字节而不是单字节，否则后续指令会整体错位。
