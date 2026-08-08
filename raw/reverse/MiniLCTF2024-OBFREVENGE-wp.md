# miniLCTF 2024 OBF_REVENGE Writeup

## 题目简述

这是 `OLLessVM` 的独立 Revenge 题，而不是同一题的附注。Windows 程序通过 `ReadConsoleW` 读取输入，先按 16 字节表异或，再把 32 字节解释为 8 个小端 DWORD，执行 12 轮环形混合；最后反转字节并再次异或后与常量比较。反编译结果用 `x - ~y - 1` 混淆加法。

## 解题过程

### 化简加法并识别轮结构

二进制中反复出现：

$$x-\mathord{\sim}y-1=x+y\pmod {2^{32}}.$$

化简后，每轮对 8 个 DWORD 顺序更新，混合左右相邻字、轮状态 `sum`、`key` 和固定的 4 DWORD 密钥。它类似 TEA 的移位、异或和轮常量结构，但不是标准 TEA，不能直接套库。

因为加密在一轮内按 `i=0..7` 原地更新，解密必须按 `i=7..0` 逆序；12 轮也必须从第 11 轮倒退。所有运算都截断为 32 位。

### 完整解密脚本

```python
from struct import pack, unpack

MASK = 0xffffffff
lkey = bytes([
    0x01, 0x02, 0x04, 0x08, 0x10, 0x20, 0x40, 0x80,
    0xff, 0xfe, 0xfc, 0xf8, 0xf0, 0xe0, 0xc0, 0x80,
])
ans = bytes([
    0x4b,0xa0,0x0c,0xff,0xab,0x0a,0x13,0xb0,
    0x32,0x91,0x6d,0x87,0x8b,0xab,0xf5,0xa5,
    0xdc,0x77,0xd4,0x95,0xb9,0x02,0xa6,0xac,
    0xe4,0x74,0x2c,0x6b,0xeb,0xe1,0x5e,0x25,
])

# 撤销加密末尾的反转与异或。
buf = bytes(ans[31-i] ^ lkey[(31-i) % 16] ^ 0x18 for i in range(32))
v = list(unpack("<8I", buf))
kword = list(unpack("<4I", lkey))

sum_ = 0xBADECADA
key = 0x2EB7B2B6
for round_ in range(12):
    sum_ = (sum_ + 0xBADECADA) & MASK
    key = (key + 0xEEB7B2B6 + (round_ % 2 == 0)) & MASK

for round_ in range(11, -1, -1):
    sum_ = (sum_ - 0xBADECADA) & MASK
    key = (key - 0xEEB7B2B6 - (round_ % 2 == 0)) & MASK

    for i in range(7, -1, -1):
        left = v[(7 + i) % 8]
        right = v[(1 + i) % 8]
        l = ((left >> 5) ^ ((right << 2) & MASK)) & MASK
        r = ((right >> 3) ^ ((left << 4) & MASK)) & MASK
        mix = (((right ^ sum_) + (kword[(key ^ i) & 3] ^ left)) & MASK)
        mix = (mix ^ ((l + r) & MASK)) & MASK
        v[i] = (v[i] - mix) & MASK

raw = pack("<8I", *v)
flag = bytes(raw[i] ^ lkey[i % 16] for i in range(32))
print(flag.decode())
```

本地运行输出：

```text
miniLCTF{Aru5T3d_h3ll_REVERSERS}
```

## 方法总结

本题的重点是先做代数化简，再严格逆转原地更新顺序。看到 TEA 风格常量并不意味着可直接调用 TEA 解密；应逐项确认相邻索引、轮常量更新时机和字节序。Python 实现还必须在加、减、左移后用 `& 0xffffffff` 模拟 DWORD 溢出。
