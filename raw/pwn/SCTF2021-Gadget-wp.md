# Gadget

## 题目简述

目标是 OLLVM 编译、musl 静态链接的 x86-64 ELF，代码地址固定但常用 ROP gadget 很不规整。程序把输入读入 0x20 字节栈缓冲区，存在栈溢出；seccomp 只允许 64 位系统调用号 0、5、37，即 `read`、`fstat`、`alarm`，同时没有可用的 `write`。利用需要完成三件事：把长 ROP 链迁移到 `.bss`，借兼容模式调用 32 位 `open`，最后用 `alarm` 的进程存活时间逐字节外带文件内容。

## 解题过程

### 第一阶段迁移到 BSS

首段输入空间较短，先利用程序中现成的寄存器设置片段调用：

```text
read(0, bss, 0x300)
```

随后设置 `rbp=bss`，执行合适的 `leave` 序列，把栈切到第二段。附件二进制还提供 `pop rsp` 路线；无论选哪条，都必须逐条检查非标准 gadget 的副作用，例如 `call r14` 会额外压入返回地址，应把 `r14` 指向可安全返回的短 gadget。

### 用 32 位 open 穿过 seccomp

过滤器只比较系统调用号，没有核对架构。x86-64 的系统调用 5 是允许的 `fstat`，而通过 `int 0x80` 进入 i386 ABI 时，编号 5 代表 `open`。先用 `retfq` 把代码段选择子切到 32 位兼容模式，设置：

```text
eax = 5
ebx = address_of("./flag")
ecx = 0
edx = 0
int 0x80
```

打开文件后，再以 `retfq` 和 64 位代码段选择子返回 long mode。新描述符通常为 3，随后用允许的 x86-64 `read(3, bss, 0x40)` 读入 flag。题目提供 `./test`，内容是字节 `01 02 03 04 05 06`，应先用它校准架构切换、文件描述符和计时链，再改为真实文件名。

### alarm 时间侧信道

由于 `write` 被禁用，附件特意保留如下 gadget：

```asm
mov bl, byte ptr [rsi + rax]
mov rdi, rbx
push r14
ret
```

令 `rsi` 指向读入缓冲区、`rax` 为字符下标、`rbx=0`、`r14=alarm`。gadget 会把当前字节装入 `bl`，从而令 `rdi` 等于 0 至 255 的字节值，再调用 `alarm(rdi)`，最后跳入死循环。远端连接在对应秒数后被信号终止，客户端测量存活时间即可恢复该字节：

```python
import time

def recover_byte(index):
    io = start_connection()
    io.send(build_stage_one())
    io.send(build_stage_two(index, filename=b"./flag\x00"))
    started = time.monotonic()
    io.recvall(timeout=300)
    elapsed = time.monotonic() - started
    io.close()
    return round(elapsed)
```

各字节互不依赖，可以并发建立连接；但应限制并发量并先对 `./test` 验证测得的 1 至 6 秒没有固定偏移。完整结果为：

```text
SCTF{woww0w_y0u_1s_g4dget_m45ter}
```

## 方法总结

本题同时利用了 syscall 编号在 x86-64 与 i386 ABI 中含义不同、seccomp 未校验架构，以及 `alarm` 可作为无输出环境中的时间通道。利用链为“短栈溢出读取长链到 BSS → `retfq` 切 32 位用编号 5 打开文件 → 回 64 位读取 → `alarm(flag[i])` 编码连接寿命”。非标准 gadget 应按“功能、必然副作用、下一次控制”三部分逐条审核，不能只看反汇编中包含了目标指令。
