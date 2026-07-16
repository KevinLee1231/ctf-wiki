# week3re1

## 题目简述

程序把输入按 4 字节小端序拆成 12 个 32 位整数，再计算 12 条带符号加减的线性表达式，并与内置常量逐项比较。目标是恢复 48 字节输入缓冲区中的 flag；运算使用 32 位整数，因此溢出回绕也是约束的一部分。

## 解题过程

反编译后记录 12 组系数和结果常量。若直接使用普通整数线性方程，会忽略 C 语言 32 位运算的模 $2^{32}$ 回绕；用 Z3 的 `BitVec(32)` 建模可以自动保留这一语义。内置结果里部分负数被反编译器显示成 `0xFFFF...` 的 64 位符号扩展值，加入约束前应明确截取低 32 位。

```python
from z3 import *

enc = [
    0x00001F42DAE73622, 0x0000C62A244D385E, 0x0000093859C830AA,
    0x000049CCE09D7B22, 0x0000E896324CBC7C, 0xFFFF74C70500B432,
    0x00003F15C47B1BA7, 0xFFFF39A40220AA20, 0x0000538009F47F6A,
    0x00002DCD82B4A8D7, 0xFFFFC90F296B929B, 0x00008F2B961DA1C9
]

flag = [BitVec("_x%d" % i, 32) for i in range(12)]
res = [0] * 12
s = Solver()

res[0] = 0x2022 - 0x175e * flag[0] - 0x9d08 * flag[1] - 0x8e48 * flag[2] + 0x4914 * flag[3] + 0xc17a * flag[4] - 0xd87d * flag[5] - 0x1d61 * flag[6] + 0x0fe4 * flag[7] + 0xbe60 * flag[8] + 0x6af0 * flag[9] - 0xe701 * flag[10] + 0x4784 * flag[11]
res[1] = 0x2022 + 0x9a8b * flag[0] + 0xe580 * flag[1] - 0xede5 * flag[2] - 0x212a * flag[3] + 0x26be * flag[4] + 0x3c3b * flag[5] - 0x311c * flag[6] + 0xf6b1 * flag[7] - 0xc7ef * flag[8] + 0xca68 * flag[9] - 0x747d * flag[10] + 0xc04c * flag[11]
res[2] = 0x2022 + 0x3933 * flag[0] - 0x7c2b * flag[1] - 0xc8d9 * flag[2] + 0x12df * flag[3] - 0x1acc * flag[4] + 0xa648 * flag[5] + 0xfec2 * flag[6] - 0x2825 * flag[7] - 0xec64 * flag[8] + 0x616d * flag[9] - 0x0632 * flag[10] - 0x9cc5 * flag[11]
res[3] = 0x2022 + 0x1c0b * flag[0] - 0x3b21 * flag[1] - 0xbb1f * flag[2] + 0x88b3 * flag[3] + 0xf675 * flag[4] - 0x7ba9 * flag[5] + 0x02b8 * flag[6] - 0x1c8f * flag[7] + 0xcbe9 * flag[8] + 0x317c * flag[9] + 0x19a3 * flag[10] + 0x42a5 * flag[11]
res[4] = 0x2022 + 0xd683 * flag[0] + 0x6b15 * flag[1] + 0xeb97 * flag[2] + 0x8382 * flag[3] - 0x6fae * flag[4] - 0xd2c1 * flag[5] - 0xc093 * flag[6] + 0x5f09 * flag[7] + 0xe038 * flag[8] + 0x9cab * flag[9] + 0x603e * flag[10] + 0x9bf6 * flag[11]
res[5] = 0x2022 + 0x10a3 * flag[0] - 0x7b5f * flag[1] - 0x0950 * flag[2] - 0xe772 * flag[3] - 0x8e68 * flag[4] + 0x63d8 * flag[5] - 0x780d * flag[6] + 0x2dfa * flag[7] - 0x99ff * flag[8] - 0x29ba * flag[9] - 0x42e3 * flag[10] - 0xddaf * flag[11]
res[6] = 0x2022 - 0x1954 * flag[0] + 0x6c6c * flag[1] - 0xd91e * flag[2] - 0x688c * flag[3] + 0xe9a8 * flag[4] + 0x9db2 * flag[5] + 0x61a3 * flag[6] + 0x4624 * flag[7] + 0xbefd * flag[8] - 0x7d56 * flag[9] - 0x1c83 * flag[10] + 0x7e02 * flag[11]
res[7] = 0x2022 - 0xded4 * flag[0] - 0x1505 * flag[1] + 0x5340 * flag[2] - 0x3501 * flag[3] + 0x1645 * flag[4] - 0xd5dd * flag[5] - 0xaaa2 * flag[6] + 0x13c3 * flag[7] + 0x982c * flag[8] - 0xce6c * flag[9] + 0x523f * flag[10] - 0x1cc3 * flag[11]
res[8] = 0x2022 - 0xef3f * flag[0] - 0xbe08 * flag[1] + 0xdad9 * flag[2] + 0x1466 * flag[3] - 0x2fe7 * flag[4] + 0x1b40 * flag[5] + 0xf290 * flag[6] + 0xfc98 * flag[7] - 0xabf0 * flag[8] + 0xa32a * flag[9] + 0x3c31 * flag[10] - 0x1a70 * flag[11]
res[9] = 0x2022 + 0xb186 * flag[0] + 0xf7a8 * flag[1] - 0x01f4 * flag[2] - 0xb5e5 * flag[3] - 0x91e8 * flag[4] + 0x4b69 * flag[5] + 0x228e * flag[6] + 0x3e1b * flag[7] - 0xa204 * flag[8] - 0x2c17 * flag[9] - 0x3ead * flag[10] + 0x358f * flag[11]
res[10] = 0x2022 - 0xe18f * flag[0] - 0x7da2 * flag[1] + 0xddc3 * flag[2] + 0x1a2d * flag[3] + 0x9539 * flag[4] + 0x5393 * flag[5] - 0x0bea * flag[6] - 0xde4d * flag[7] + 0x1ecc * flag[8] + 0x8259 * flag[9] + 0x95a0 * flag[10] - 0xa409 * flag[11]
res[11] = 0x2022 + 0xbdc3 * flag[0] - 0xc63e * flag[1] + 0x3257 * flag[2] + 0x2dd0 * flag[3] - 0x9e70 * flag[4] + 0xbbc7 * flag[5] - 0x2ff9 * flag[6] + 0xbab7 * flag[7] + 0xbf7b * flag[8] - 0x3041 * flag[9] - 0x96d3 * flag[10] - 0x8197 * flag[11]

for i in range(12):
    s.add(res[i] == BitVecVal(enc[i] & 0xFFFFFFFF, 32))
s.add(flag[11] == 0)

s_flag = b""
if s.check() == sat:
    out = s.model()
    for i in range(12):
        s_flag += out[flag[i]].as_long().to_bytes(4, byteorder='little')
    print(s_flag.rstrip(b"\x00").decode())
else:
    print("Wrong")

# flag{1853dd27-8720-48c8-a725-1fe52b8b25e7}
```

求解结果为：

```text
flag{1853dd27-8720-48c8-a725-1fe52b8b25e7}
```

## 方法总结

本题本质是固定宽度整数上的线性约束求解。关键不是“看到方程就上 Z3”，而是先确认变量宽度、字节序和溢出规则：12 个变量分别对应 4 字节小端序数据，所有表达式在 $\mathbb Z/2^{32}\mathbb Z$ 上计算。末尾零值用于约束 C 字符串缓冲区的填充，解出后再去掉 `\x00`，不能在建模前擅自缩短输入。
