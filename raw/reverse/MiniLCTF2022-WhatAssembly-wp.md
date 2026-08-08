# MiniLCTF2022 WhatAssembly Writeup

## 题目简述

附件是 WebAssembly 程序，实现了一个按字节运算、结构类似 ChaCha 的 16 字节状态变换。固定密文共 80 字节，分成 5 个状态块；每块逆 42 轮后，后 8 字节依次拼成 flag。前 8 字节只是状态前缀或上一块关联数据，并非最终明文的一部分。

## 解题过程

把 WASM 转成 C 或直接分析线性内存访问，可还原四元组操作。单次逆操作必须按下面的顺序更新，因为后一步使用的是前一步已经恢复的状态值：

```python
def rotl8(value, bits):
    return ((value << bits) | (value >> (8 - bits))) & 0xff

def quarou_rev(s, a, b, c, d):
    s[a] ^= rotl8((s[d] + s[c]) & 0xff, 1)
    s[c] ^= rotl8((s[b] + s[a]) & 0xff, 3)
    s[d] ^= rotl8((s[c] + s[b]) & 0xff, 2)
    s[b] ^= rotl8((s[a] + s[d]) & 0xff, 4)
```

每一轮包含八次 `quarou_rev`，调用顺序如下；对每个 16 字节密文块重复 42 轮：

```python
order = [
    (14, 9, 4, 3), (13, 8, 7, 2),
    (12, 11, 6, 1), (15, 10, 5, 0),
    (15, 11, 7, 3), (14, 10, 6, 2),
    (13, 9, 5, 1), (12, 8, 4, 0),
]

flag = bytearray()
for offset in range(0, len(enc), 16):
    state = list(enc[offset:offset + 16])
    for _ in range(42):
        for args in order:
            quarou_rev(state, *args)
    flag.extend(state[8:])

print(bytes(flag).rstrip(b"\x00"))
```

本地重放得到各块后半部分依次为 `miniLctf`、`{0ooo00o`、`h!h3ll0_`、`WASM_h4c`、`k3r!}`，最终结果是：

```text
miniLctf{0ooo00oh!h3ll0_WASM_h4ck3r!}
```

## 方法总结

WASM 逆向的重点是把线性内存地址还原成状态数组索引，并严格保持更新顺序。虽然常量 `D33.B4T0` 看似密钥，它只用于初始状态填充，实际没有形成保密的密钥调度。逐块打印完整 16 字节状态是很有效的自检方式：第一块应出现 `D33.B4T0miniLctf`，否则通常是旋转位数、字节掩码或调用顺序写错。
