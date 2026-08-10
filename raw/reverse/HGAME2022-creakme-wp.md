# creakme

## 题目简述

程序包含一张类似 Base64 的字符串表，但实际校验是按两个 32 位整数为一组执行 32 轮魔改 TEA。与标准 TEA 相比，它使用 `0x12345678` 作为 delta，并在每轮更新中额外异或当前的 `sum`。需要严格模拟 32 位无符号溢出，再按小端序还原明文。

## 解题过程

密钥来自字符串 `ABCDEFGHIJKLMNOP...` 的前 16 字节，按小端 DWORD 解释为：

```python
key = [0x44434241, 0x48474645, 0x4c4b4a49, 0x504f4e4d]
```

程序用于比较的 8 个 DWORD 为：

```python
target = [
    0x48d93488, 0x030c144c, 0x52eb78c2, 0xed9ce5ed,
    0xae1fede6, 0xba5a126d, 0xcf9284aa, 0x65e0f2e3,
]
```

逆向时必须在每次加减后截断到 32 位。解密脚本如下：

```python
import struct

MASK = 0xffffffff
DELTA = 0x12345678
KEY = [0x44434241, 0x48474645, 0x4c4b4a49, 0x504f4e4d]
words = [
    0x48d93488, 0x030c144c, 0x52eb78c2, 0xed9ce5ed,
    0xae1fede6, 0xba5a126d, 0xcf9284aa, 0x65e0f2e3,
]

for offset in range(0, len(words), 2):
    left, right = words[offset:offset + 2]
    total = (DELTA * 33) & MASK

    for _ in range(32):
        total = (total - DELTA) & MASK
        right = (
            right
            - (
                total
                ^ ((left << 4) + KEY[0])
                ^ (left + total)
                ^ ((left >> 5) + KEY[1])
            )
        ) & MASK
        left = (
            left
            - (
                total
                ^ ((right << 4) + KEY[2])
                ^ (right + total)
                ^ ((right >> 5) + KEY[3])
            )
        ) & MASK

    words[offset:offset + 2] = left, right

plaintext = struct.pack("<8I", *words).rstrip(b"\x00")
print(plaintext.decode())
```

输出为：

```text
hgame{H4ppy_v4c4ti0n!}
```

原 PDF 只展示了算法框架；目标 DWORD 与小端序结果通过 [Second_BC 的 HGAME2022 Reverse 复现](https://secondbc.github.io/SecondBC/2022/02/22/Hgame2022-ReverseWriteUp/) 交叉核对。上述正文已包含外部记录中用于复现的全部关键常量。

## 方法总结

识别 TEA 变种时不能只套标准模板，要逐项核对 delta、更新顺序、密钥索引和额外异或项。C 代码中的 `unsigned int` 会自然回绕，而 Python 整数不会，因此每轮都要 `& 0xffffffff`。最后还要按目标平台的小端字节序打包 DWORD，否则会得到每四字节倒置的字符串。
