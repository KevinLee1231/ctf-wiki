# Miversc

## 题目简述

附件 `miverse.txt` 由大量 `Ook.`、`Ook?` 和 `Ook!` 组成，这是 Ook! 深奥语言。Ook! 每两个词对应一条 Brainfuck 指令，题目需要先完成语法转换，再从 Brainfuck 的定值运算中恢复被编码的输入。

## 解题过程

Ook! 与 Brainfuck 的完整映射如下：

| Ook! 词对 | Brainfuck |
| --- | --- |
| `Ook. Ook?` | `>` |
| `Ook? Ook.` | `<` |
| `Ook. Ook.` | `+` |
| `Ook! Ook!` | `-` |
| `Ook! Ook.` | `.` |
| `Ook. Ook!` | `,` |
| `Ook! Ook?` | `[` |
| `Ook? Ook!` | `]` |

可以直接将附件转换为 Brainfuck：

```python
import re
from pathlib import Path

mapping = {
    ("Ook.", "Ook?"): ">",
    ("Ook?", "Ook."): "<",
    ("Ook.", "Ook."): "+",
    ("Ook!", "Ook!"): "-",
    ("Ook!", "Ook."): ".",
    ("Ook.", "Ook!"): ",",
    ("Ook!", "Ook?"): "[",
    ("Ook?", "Ook!"): "]",
}

tokens = re.findall(r"Ook[.!?]", Path("miverse.txt").read_text())
assert len(tokens) % 2 == 0
brainfuck = "".join(
    mapping[(tokens[index], tokens[index + 1])]
    for index in range(0, len(tokens), 2)
)
print(brainfuck)
```

转换后的程序先输出提示语，随后连续读取 25 个字符。每个输入字符都会经过一段固定减法，使题目预期字符归零。例如：

```brainfuck
,<++++++[->--------<]>
```

循环从第一个输入字节减去 $6\times8=48$，因此对应字符是 ASCII 48，即 `0`；下一段减去 $10\times12=120$，对应字符是 `x`。按相同方法依次还原 25 段常量，可得：

```text
0xGame{Just_Reverse_OoK!}
```

需要注意，程序末尾输出的 `flag is what you input` 并不会真正判断输入是否正确；flag 信息实际藏在每次读入后的算术常量中，不能只根据运行输出下结论。

## 方法总结

本题包含两层表示：Ook! 词对先映射为 Brainfuck 指令，Brainfuck 中的循环乘法再表示各字符的 ASCII 常量。完整列出映射并分析输入后的定值减法，就能在不依赖在线解释器的情况下复现 flag。
