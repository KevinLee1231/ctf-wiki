# Crackme

## 题目简述

这是一个 C++ 异常处理题。主算法没有以普通控制流完整呈现，而是连续抛出三次异常，把每轮运算拆到不同的 catch handler 中。三个 handler 分别完成变形 XTEA 的三步：更新 `sum`、更新右半块 `v1`、更新左半块 `v0`。

程序使用 4 个 64 位分组、固定 4 项密钥和 32 轮运算。还原异常控制流后，对给定 8 个 `uint32_t` 执行逆运算即可得到 flag。

## 解题过程

在 IDA 中跟进异常处理表和各个 catch handler，可以看到处理逻辑虽然被拆散，但每个 handler 都会回到同一轮循环。若反编译器无法自动恢复，可在 handler 起始地址先按 `U` 取消错误定义，再按 `P` 建立函数；生成的伪代码不够美观，但足以识别运算。

合并三个 handler 后，每轮逻辑为：

```c
sum ^= 0x33221155;
v1 -= ((((v0 << 5) ^ (v0 >> 6)) + v0)
       ^ (sum + key[(sum >> 11) & 3]));
v0 -= ((((v1 << 4) ^ (v1 >> 5)) + v1)
       ^ (sum + key[sum & 3]));
```

所有运算均在 32 位无符号整数上回绕。程序在正式循环前还把 `sum` 与 `delta` 异或 32 次；因为次数为偶数，最终仍有 `sum=0`。下面的 Python 脚本显式限制到 32 位，并按 Windows/x86 小端序拼回字节：

```python
from struct import pack

MASK = 0xFFFFFFFF
DELTA = 0x33221155
KEY = [1234, 2345, 3456, 4567]
CIPHERTEXT = [
    855388650, 4032196418,
    4177899698, 1598378430,
    4215209147, 1802165040,
    75733113, 792951007,
]


def decrypt_pair(v0, v1):
    total = 0
    for _ in range(32):
        total ^= DELTA

    for _ in range(32):
        total ^= DELTA

        mix = ((((v0 << 5) & MASK) ^ (v0 >> 6)) + v0) & MASK
        v1 = (v1 - (mix ^ ((total + KEY[(total >> 11) & 3]) & MASK))) & MASK

        mix = ((((v1 << 4) & MASK) ^ (v1 >> 5)) + v1) & MASK
        v0 = (v0 - (mix ^ ((total + KEY[total & 3]) & MASK))) & MASK

    return v0, v1


words = []
for offset in range(0, len(CIPHERTEXT), 2):
    words.extend(decrypt_pair(CIPHERTEXT[offset], CIPHERTEXT[offset + 1]))

plaintext = b"".join(pack("<I", word) for word in words).rstrip(b"\x00")
print(plaintext.decode())
```

输出为：

```text
hgame{C_p1us_plus_exc3pti0n!!!!}
```

## 方法总结

- C++ 异常处理不仅用于错误恢复，也能被用作隐藏真实控制流；应把异常表、throw 点和 catch handler 一起分析。
- 不必把整个程序修复到能完美反编译，只要逐个提取 handler 的状态更新并按执行顺序组合即可。
- TEA/XTEA 家族高度依赖固定位宽回绕；用 Python 重写时必须在每次关键运算后与 `0xffffffff`，并按原平台端序还原字节。
- 本题并非标准 XTEA：`delta`、移位常数和 `sum` 更新方式均有变化，直接套现成实现不会得到正确结果。
