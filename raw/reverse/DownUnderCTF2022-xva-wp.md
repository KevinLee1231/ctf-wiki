# DownUnderCTF 2022 xva Writeup

## 题目简述

程序读取 32 字节，并将其解释成 16 个小端有符号 16 位整数。第一项检查要求这些整数之和为 384068；第二项使用 AVX2 的 16 位加法、32 位通道置换、低 16 位乘法和减法，最后把结果与八个 32 位常量比较。目标是准确模拟向量通道顺序并求解输入。

## 解题过程

反编译中真正参与计算的指令对应 `_mm256_add_epi16`、`_mm256_permutevar8x32_epi32`、`_mm256_mullo_epi16` 和 `_mm256_sub_epi16`。若原 16 位通道为 $s_i$，先计算
$r_i=(s_i+0x419b)\bmod2^{16}$，再对由两个 16 位通道组成的 32 位通道做两次置换，得到 $c_i$ 和 $d_i$，最终满足
$o_i=(r_i\cdot d_i-c_i)\bmod2^{16}$。

最容易出错的是 `_mm256_set_epi16` 和 `_mm256_set_epi32` 的参数顺序：源码参数按高通道到低通道书写。因此 16 位向量的低到高顺序是 `input_[15]` 到 `input_[0]`；两组 32 位置换索引按低到高分别为 `[3,1,0,6,7,4,3,1]` 和 `[1,0,3,2,6,7,4,5]`。

用 Z3 的固定位宽位向量可原样保留模 $2^{16}$ 语义，无需人为修正溢出字节：

```python
from z3 import *

b = [BitVec(f'b{i}', 8) for i in range(32)]
s16 = [Concat(b[2*i + 1], b[2*i]) for i in range(16)]
vec = list(reversed(s16))
res = [x + BitVecVal(0x419b, 16) for x in vec]

lanes32 = [Concat(res[2*i + 1], res[2*i]) for i in range(8)]
p1 = [3, 1, 0, 6, 7, 4, 3, 1]
p2 = [1, 0, 3, 2, 6, 7, 4, 5]

def split_words(values):
    words = []
    for value in values:
        words += [Extract(15, 0, value), Extract(31, 16, value)]
    return words

c = split_words([lanes32[i] for i in p1])
d = split_words([lanes32[i] for i in p2])
out = [res[i] * d[i] - c[i] for i in range(16)]
packed = [Concat(out[2*i + 1], out[2*i]) for i in range(8)]

answers = [
    0x85765e6f, 0x7b761fa8, 0x05306ec9, 0xbd5d8cfa,
    0xc2db0af6, 0x6cf52153, 0xabec2bcd, 0x5c211278,
]

solver = Solver()
for value, answer in zip(packed, answers):
    solver.add(value == answer)
solver.add(Sum([BV2Int(x, is_signed=True) for x in s16]) == 384068)
for i, ch in enumerate(b'DUCTF{'):
    solver.add(b[i] == ch)
for ch in b:
    solver.add(UGE(ch, 0x20), ULE(ch, 0x7e))

assert solver.check() == sat
model = solver.model()
print(bytes(model.eval(ch).as_long() for ch in b).decode())
```

模型给出的 32 字节输入为：

```text
DUCTF{A_V3ry_eXc3ll3n7_r3v3rs3r}
```

## 方法总结

本题难点在 SIMD 数据布局，而不在单条指令。要同时追踪三种顺序：输入字节到小端 16 位整数、`set` intrinsic 的高到低参数顺序、以及 32 位置换后再拆回两个 16 位通道的顺序。用 8/16/32 位 BitVec 建模能自然表达截断和溢出，比先提升到普通整数再手工取模更不易出错；和校验及 `DUCTF{` 前缀则用于选出可打印的正确解。
