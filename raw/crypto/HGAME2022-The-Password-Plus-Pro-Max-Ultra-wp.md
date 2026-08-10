# The Password Plus Pro Max Ultra

## 题目简述

题目把 flag 切成若干个不超过 64 位的整数块。每块都经过一组异或与 64 位循环左移：

$$
y=x\oplus\operatorname{ROL}(x,k_1)\oplus\cdots\oplus\operatorname{ROL}(x,k_t)
$$

所有运算都可以视为 $GF(2)$ 上的 64 维线性变换。每组移位数的个数 $t$ 都是偶数，因此该变换的 64 次幂是恒等变换，可以不用矩阵求逆，直接构造逆变换。

## 解题过程

令 $R^k$ 表示循环左移 $k$ 位，线性算子为：

$$
T=I+R^{k_1}+R^{k_2}+\cdots+R^{k_t}
$$

在特征为 2 的域上，各个循环移位算子彼此可交换，平方时的交叉项成对抵消：

$$
T^2=I+R^{2k_1}+R^{2k_2}+\cdots+R^{2k_t}
$$

连续平方 6 次后，所有移位量都乘以 $64$。64 位循环移位满足 $R^{64k}=I$，而 $t$ 为偶数，所以：

$$
T^{64}=I+tI=I
$$

于是 $T^{-1}=T^{63}$。不必真的循环 63 次；因为 $63=1+2+4+8+16+32$，依次使用原移位列表、两倍移位列表、四倍移位列表，执行 6 次即可得到 $T^{63}y$。

下面的脚本包含题目给出的全部 13 组密文和移位参数，不依赖 SageMath 或 `libnum`：

```python
from functools import reduce
from operator import xor

MASK = (1 << 64) - 1

CIPHERS = [
    2656224875120172108,
    1261711348908201279,
    18219282869614004824,
    15279054981769814589,
    7966355346882200701,
    5641592208539483808,
    1502927090219059154,
    3996223120734273799,
    18295033054788808618,
    18126228466291248047,
    9413762634844369954,
    8964324149921197550,
    6962485320551449848,
]

SHIFTS = [
    [8, 35],
    [19, 29, 30, 45],
    [6, 16, 18, 21, 44, 55],
    [10, 26, 30, 46, 51, 54, 58, 63],
    [5, 13, 25, 29, 37, 39, 43, 52, 53, 59],
    [1, 26, 31, 39, 40, 41, 43, 45, 49, 52, 54, 62],
    [8, 12, 19, 20, 30, 32, 34, 40, 41, 45, 46, 49, 55, 58],
    [2, 3, 5, 6, 8, 10, 15, 19, 26, 27, 33, 40, 42, 47, 52, 61],
    [1, 16, 17, 27, 28, 30, 32, 36, 37, 38, 39, 48, 49, 51, 55, 57, 59, 62],
    [5, 11, 12, 20, 22, 23, 25, 27, 31, 32, 33, 37, 44, 45, 49, 52, 53, 59, 61, 62],
    [2, 7, 10, 12, 18, 19, 20, 22, 26, 29, 33, 34, 38, 40, 41, 45, 46, 51, 54, 56, 57, 60],
    [3, 4, 5, 9, 12, 13, 18, 19, 21, 23, 24, 25, 30, 33, 34, 35, 37, 39, 43, 44, 46, 49, 50, 53],
    [1, 3, 6, 7, 10, 11, 13, 14, 23, 27, 32, 33, 35, 37, 39, 41, 46, 48, 49, 50, 51, 53, 54, 56, 58, 62],
]

def rol64(value: int, shift: int) -> int:
    shift &= 63
    if shift == 0:
        return value & MASK
    return ((value << shift) | (value >> (64 - shift))) & MASK

def transform(value: int, shifts: list[int]) -> int:
    rotated = (rol64(value, shift) for shift in shifts)
    return value ^ reduce(xor, rotated)

def decrypt(value: int, shifts: list[int]) -> int:
    current_shifts = shifts[:]
    for _ in range(6):
        value = transform(value, current_shifts)
        current_shifts = [2 * shift for shift in current_shifts]
    return value

plaintext = bytearray()
for cipher, shifts in zip(CIPHERS, SHIFTS):
    block = decrypt(cipher, shifts)
    length = max(1, (block.bit_length() + 7) // 8)
    plaintext.extend(block.to_bytes(length, "big"))

print(plaintext.decode())
```

这里按每个整数的最短大端表示拼接，等价于原题解的 `libnum.n2s()`；若固定写成 8 字节，靠后的短块会在 flag 中间引入多余的零字节。输出为：

```text
hgame{XOr|RoR&rOl|Is+vERY#coMmon*BiTwisE$OPeraTiOn*IT@is%oFten,ENCOUntErED*in.syMMeTRic?encryPtION}
```

## 方法总结

异或和循环移位都是 $GF(2)$ 上的线性操作，适合用线性算子或矩阵统一分析。此题最简洁的逆不是逐位猜测，而是利用特征 2 下的平方性质和 64 位旋转周期，证明 $T^{64}=I$，再计算 $T^{63}$。实现时还要注意位宽、旋转方向和整数转字节的端序，否则数学正确也会得到错位文本。
