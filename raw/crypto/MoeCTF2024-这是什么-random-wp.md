# 这是什么？random！

## 题目简述

程序要求连续猜中十个五位数。看似需要预测随机数，实际却把 `localtime(...)->tm_yday` 直接作为 `srandom` 的种子；该字段是从 0 开始的一年内序号，因此同一天内种子固定，后续 `random()` 序列完全可复现。

## 解题过程

Python 的 `time.localtime().tm_yday` 从 1 开始，而 C 的 `struct tm.tm_yday` 从 0 开始，所以本地应使用 `tm_yday - 1`。再通过 `ctypes` 调用与服务端一致的 glibc `srandom/random`，逐项发送 `random() % 90000 + 10000`：

```python
from ctypes import CDLL
from time import localtime
from pwn import remote

io = remote("host", 10000)
libc = CDLL("libc.so.6")
libc.srandom(localtime().tm_yday - 1)

for _ in range(10):
    guess = libc.random() % 90000 + 10000
    io.sendlineafter(b"\n", str(guess).encode())

# 后两次输入不参与随机数校验，任意整数即可。
io.sendlineafter(b"\n", b"42")
io.sendlineafter(b"\n", b"42")
io.interactive()
```

若本地与远端时区不同，或恰逢远端跨过午夜，应同时尝试相邻的两个 `tm_yday`，而不是把失败误判为 libc 算法不同。

## 方法总结

本题的漏洞是可预测种子，而不是 `random()` 本身。复现时必须同时匹配随机数实现、种子含义和取模方式；Python 与 C 同名日期字段的起始下标差一，是最容易遗漏的细节。
