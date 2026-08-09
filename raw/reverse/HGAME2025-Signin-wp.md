# Signin

## 题目简述

程序表面上是一个 XXTEA 校验器，但密钥和轮常量都与运行时状态绑定：

- 程序读取调试寄存器，将其值参与 `delta` 的生成；正常运行时相关寄存器为 `0`，所以实际 `delta = 0`。
- 程序复制从 `main` 附近开始的 `0x10000` 字节代码，并按块计算 CRC32，四个校验值组成 XXTEA 的 128 位密钥。

普通软件断点会把目标指令改成 `int3`，导致 CRC32 随之改变；硬件断点虽然不修改代码，却会占用调试寄存器。因此需要同时处理这两处反调试影响，才能取得正确密钥并还原明文。

## 解题过程

静态分析可以识别出 XXTEA 的 `MX` 结构和 9 个 32 位整数组成的密文。程序的解密轮数为：

```text
6 + 52 / 9 = 11
```

调试时不要在被 CRC32 覆盖的代码区下软件断点。可在 CRC32 计算结束之后下硬件断点，等程序把调试寄存器读入全局变量后，再把代表 `delta` 的全局值改回 `0`。随后读取四个 CRC32 结果，得到密钥：

```text
0x97A25FB5  0xE1756DBA  0xA143464A  0x5A8F284F
```

密文为：

```text
0x3050EA23  0x47514C00  0x2B769CEE
0x1794E6D5  0xB3E42BED  0x61D536CB
0x7CA0C2C0  0x5ED767FE  0xC579E0AF
```

由于本题运行时的 `delta` 是 `0`，每轮的 `sum` 始终不变。下面的 Python 代码按无符号 32 位整数模拟题目中的 XXTEA 解密，并按小端序重新拼接字节：

```python
import struct

MASK = 0xFFFFFFFF

key = [
    0x97A25FB5,
    0xE1756DBA,
    0xA143464A,
    0x5A8F284F,
]

data = [
    0x3050EA23,
    0x47514C00,
    0x2B769CEE,
    0x1794E6D5,
    0xB3E42BED,
    0x61D536CB,
    0x7CA0C2C0,
    0x5ED767FE,
    0xC579E0AF,
]

n = len(data)
rounds = 6 + 52 // n
sum_value = 0
y = data[0]

for _ in range(rounds):
    e = (sum_value >> 2) & 3

    for p in range(n - 1, 0, -1):
        z = data[p - 1]
        mx = (
            (((z >> 5) ^ ((y << 2) & MASK))
             + ((y >> 3) ^ ((z << 4) & MASK)))
            ^ ((sum_value ^ y) + (key[(p & 3) ^ e] ^ z))
        ) & MASK
        data[p] = (data[p] - mx) & MASK
        y = data[p]

    z = data[n - 1]
    mx = (
        (((z >> 5) ^ ((y << 2) & MASK))
         + ((y >> 3) ^ ((z << 4) & MASK)))
        ^ ((sum_value ^ y) + (key[e] ^ z))
    ) & MASK
    data[0] = (data[0] - mx) & MASK
    y = data[0]

plaintext = struct.pack("<9I", *data).decode()
print(plaintext)
```

输出为：

```text
3fe4722c-1dbf-43b7-8659-c1c4a0e42e4d
```

结合程序要求的 `hgame{...}` 输入格式，flag 为：

```text
hgame{3fe4722c-1dbf-43b7-8659-c1c4a0e42e4d}
```

原 PDF 没有展示最终运行输出；这里的结果由上述常量本地复算得到，并与[公开选手复盘](https://www.cnblogs.com/T0fV404/p/18829400)中的结果交叉核对。

## 方法总结

本题的难点是两种反调试手段相互配合：软件断点会破坏代码 CRC32，硬件断点又会改变调试寄存器。正确做法是用硬件断点跨过代码完整性检查，再把已读取的寄存器派生值恢复成非调试状态的 `0`，从内存中取得四个正确 CRC32 密钥。识别 XXTEA 后还不能直接套标准实现，因为本题把 `delta` 置零；必须严格复现目标程序的无符号溢出、轮数和小端字节序。
