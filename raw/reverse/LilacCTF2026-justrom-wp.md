# JustROM

## 题目简述

题目是一个 SPARC 32-bit big-endian ROM 程序，基址为 `0x40004000`。源码中定义了内存映射输入、输出和命令寄存器：

```c
#define INPUT   ((volatile char *)0x42000000)
#define RESULT  ((volatile char *)0x42000100)
#define COMMAND ((volatile unsigned char *)0x42000200)
```

程序通过 `COMMAND` 操作 `INPUT` 缓冲区，最后执行 `check()`。`check()` 使用固定 key、nonce 和 ChaCha-like block 生成 keystream，再校验输入。

## 解题过程

### 关键观察

程序先把 `expected` 与固定 `xor` 数组异或，然后生成 ChaCha keystream，再把 `INPUT` 与 keystream 异或后和 `expected` 比较：

```c
for (i = 0; i < 32; i++)
    expected[i] ^= xor[i];

chacha_block(stream, state);

for (i = 0; i < 32; i++)
    INPUT[i] ^= ((u8*)stream)[i];

if (INPUT[i] != expected[i])
    goto fail;
```

所以正确输入满足：

$$
input = expected \oplus xor \oplus stream
$$

### 求解步骤

从源码取出 key、nonce、counter 和 ChaCha 常量：

```c
key = 11223344 55667788 99aabbcc ddeeff00 deadbeef cafebabe 0badf00d 13371337
nonce = 41414141 42424242 43434343
counter = 1
```

按源码的 8 轮 ChaCha block 计算 keystream，再代入上面的异或关系，得到输入：

```text
LilacCTF{d0ntl@@kl1kechch4atall}
```

在模拟器中通过 `COMMAND` 把这 32 字节写入 `INPUT`，再发送 `0xEE` 触发 `check()`，`RESULT` 会循环写入 `Win`。

## 方法总结

- 核心技巧：读懂内存映射 ROM 的输入模型，把校验逻辑还原为固定 keystream 异或。
- 识别信号：嵌入式/ROM 题中出现 `INPUT`、`RESULT`、`COMMAND` 这类固定地址，通常需要先建立外设交互模型，再看真正的密码校验。
- 复用要点：ChaCha 变体题要确认轮数、counter、nonce 端序和最终加回原 state 的方式；这些细节错一个字节，最终输入都会完全不同。
