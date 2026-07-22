# 特工 luo：深入敌营

## 题目简述

附件是 UPX 压缩的 Windows x64 程序。脱壳后，`main` 并不直接校验输入，而是解释一段自定义字节码；需要恢复虚拟机状态、指令语义和条件分支，再用 8 位 Z3 位向量求满足成功路径的可打印输入。

## 解题过程

先执行 `upx -d agent-luo-1.exe`，再分析脱壳后的 `main`。解释器维护四项核心状态：

- `_code`：虚拟指令指针；
- `save` / `_rsp`：以字节为单位的虚拟栈指针；
- `stack`：虚拟栈内存；
- `saver`：一字节临时寄存器。

`switch (insn[_code])` 中可确认的主要 opcode 如下：

| opcode | 语义 |
| --- | --- |
| `0x0b`、`0x99`、`0xb5`、`0xd3` | 分别弹出 8、4、2、1 字节 |
| `0x32`、`0xb7`、`0x91`、`0xea` | 压入 byte、word、dword、qword 立即数 |
| `0x49` | 复制栈顶字节 |
| `0x0e`、`0x72` | 对栈顶按位取反 |
| `0x14` | 对两个栈顶操作数按位与 |
| `0x7b` | 按栈顶给出的长度读入用户字符串到 `Buffer` |
| `0x7c` / `0xad` | 栈顶保存到 `saver` / 把 `saver` 压栈 |
| `0x8d` | 栈顶为零时跳转，随后弹出条件和四字节目标 |
| `0x8e` | 异或解码并打印虚拟栈中的字符串 |
| `0xb8` | 以栈中地址和索引从 `Buffer` 取一个输入字节 |
| `0xfb` / `0xff` | 归一化为布尔值 / 逻辑非 |
| `0x19` | 退出虚拟机并返回后续 dword |

应把输入字符建模为 `BitVec(..., 8)`，不能使用无界 `Int`，否则 `~`、溢出和按字节截断的语义都会错误：

```python
from string import printable
from z3 import BitVec, Or, Solver

length = 54  # 不含字符串结尾的 NUL；由成功模型长度确认
chars = [BitVec(f"ch_{i}", 8) for i in range(length)]
solver = Solver()

allowed = sorted({ord(c) for c in printable if c not in "\r\n\t\x0b\x0c"})
for char in chars:
    solver.add(Or(*(char == value for value in allowed)))

for index, value in enumerate(b"moectf{"):
    solver.add(chars[index] == value)
solver.add(chars[-1] == ord("}"))
```

随后按 opcode 表解释 `insn`：常量仍用 8/16/32/64 位位向量，`0xb8` 取到相应的 `chars[index]`；遇到 `0x8d` 时，把“条件为零”和“条件非零”分别加入求解器副本，使用工作队列保留可满足分支。到达成功的 `0x19` 路径后读取模型并拼接字符。

仓库这一题只提供 `agent-luo-1.exe` 和题面，没有作者的 solver、源码或未压缩字节码，因此不能凭空补一份声称可运行的完整 exp；上面的 opcode 与符号执行步骤来自附件解释器本身，最终模型为：

```text
moectf{vv1ii?AGENT+1U0_-*7_$[TH/./e]+$_FinAL_v!c70Ry?}
```

把该字符串输入原程序可通过校验。

## 方法总结

自定义 VM 逆向应先恢复状态布局和每条 opcode 的精确宽度，再做符号执行。Z3 只是约束求解器，真正容易出错的是虚拟栈的字节序、弹栈长度、布尔归一化和条件跳转；先写具体值解释器与原程序逐步对拍，再替换成位向量最稳妥。
