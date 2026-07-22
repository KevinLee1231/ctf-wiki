# dynamic

## 题目简述

程序内置 48 字节密文，先调用同一函数解密，打印提示后又立即加密回去。参数 `4294967284` 截断为 32 位有符号数时是 `-12`，因此进入 XXTEA 解密分支；随后参数 `12` 进入加密分支。可以动态断在两次调用之间读明文，也可以按反编译结果离线解密。

## 解题过程

核心函数以 `n` 的符号选择方向：

- `n > 1`：对 `n` 个 32 位 word 加密；
- `n < -1`：令 `n = -n` 后解密；
- delta 被替换为 `0x114514`；
- key 为 `(0xcafebabe, 0xdeadc0de, 0xd3906, 0x114514)`。

密文共 48 字节，即 12 个小端 `uint32_t`。完整离线解密脚本如下：

```python
from struct import pack, unpack

MASK = 0xFFFFFFFF
DELTA = 0x114514
KEY = (0xCAFEBABE, 0xDEADC0DE, 0xD3906, 0x114514)

ciphertext = bytes([
    0xA2, 0x05, 0x69, 0x8B, 0xDA, 0x17, 0x05, 0xE1,
    0xDC, 0xCC, 0xCC, 0xC1, 0x64, 0x74, 0xFA, 0x50,
    0xD5, 0xA1, 0x9A, 0xAC, 0xDC, 0xDE, 0x64, 0xBF,
    0x94, 0x2D, 0x23, 0xF3, 0x01, 0xD5, 0x62, 0xC8,
    0xEA, 0xAD, 0xD2, 0xD6, 0x2A, 0x50, 0x5E, 0x6B,
    0x73, 0x0C, 0xFD, 0x8C, 0x3D, 0x38, 0x3D, 0xD1,
])

def mx(z, y, total, key, p, e):
    return (
        (((z >> 5) ^ (y << 2)) + ((y >> 3) ^ (z << 4)))
        ^ ((total ^ y) + (key[(p & 3) ^ e] ^ z))
    ) & MASK

def xxtea_decrypt(data):
    words = list(unpack("<12I", data))
    n = len(words)
    rounds = 6 + 52 // n
    total = (rounds * DELTA) & MASK
    y = words[0]

    while total:
        e = (total >> 2) & 3
        for p in range(n - 1, 0, -1):
            z = words[p - 1]
            words[p] = (words[p] - mx(z, y, total, KEY, p, e)) & MASK
            y = words[p]

        z = words[n - 1]
        words[0] = (words[0] - mx(z, y, total, KEY, 0, e)) & MASK
        y = words[0]
        total = (total - DELTA) & MASK

    return pack("<12I", *words)

print(xxtea_decrypt(ciphertext).rstrip(b"\x00").decode())
```

输出：

```text
moectf{18d4c944-947c-4808-9536-c7d34d6b3827}
```

动态方法则在第一次 `sub_14001129E(v7, -12, v8)` 返回后、第二次 `sub_14001129E(v7, 12, v8)` 调用前查看 `v7`，能看到相同明文。

## 方法总结

识别 XXTEA 的特征包括 `6 + 52 / n` 轮数、`(sum >> 2) & 3` 的 key 索引和相邻 word 混合。反编译器把 `0xfffffff4` 显示成 `4294967284` 容易掩盖负数分支；应根据实际参数宽度重新解释符号。
