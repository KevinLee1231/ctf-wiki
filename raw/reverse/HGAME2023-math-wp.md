# math

## 题目简述

程序读取 25 个字符，将其视为一个 $5\times5$ 矩阵，再与常量矩阵相乘并比较结果。反编译器把输入位置显示成 `savedregs - 368`，看起来像特殊寄存器变量，但根据栈偏移可知它实际指向输入缓冲区。

## 解题过程

栈帧中 `savedregs` 位于 `rsp+0x180`，所以 `savedregs-368` 即：

$$
(rsp+0x180)-0x170=rsp+0x10
$$

这正是输入数组 `v8` 的地址。把输入矩阵记为 $C$，程序内的常量矩阵记为 $B$，比较目标记为 $A$，循环实际计算：

$$
C B=A
$$

因此：

$$
C=A B^{-1}
$$

从程序中抄出两个常量矩阵后，可以直接求逆并验证乘积：

```python
import numpy as np

B = np.array([
    0x7E, 0xE1, 0x3E, 0x28, 0xD8,
    0xFD, 0x14, 0x7C, 0xE8, 0x7A,
    0x3E, 0x17, 0x64, 0xA1, 0x24,
    0x76, 0x15, 0xB8, 0x1A, 0x8E,
    0x3B, 0x1F, 0xBA, 0x52, 0x4F,
], dtype=np.int64).reshape(5, 5)

A = np.array([
    0xF9FE, 0x8157, 0x108B2, 0xD605, 0xF21B,
    0x10FF3, 0x9146, 0x11212, 0xCF76, 0x10C46,
    0xF76B, 0x77DF, 0x103BE, 0xC6F8, 0xED8A,
    0xBE90, 0x75EC, 0xEAC8, 0xAE37, 0xCC29,
    0xA828, 0x5C6C, 0xAB4A, 0x836E, 0xACEE,
], dtype=np.int64).reshape(5, 5)

C = np.rint(A @ np.linalg.inv(B)).astype(np.int64)
assert np.array_equal(C @ B, A)

plaintext = bytes(C.ravel()).rstrip(b"\x00")
print(plaintext.decode())
```

输出矩阵为：

```text
104 103  97 109 101
123 121  48 117 114
 95 109  64 116 104
 95  49 115  95 103
 79  48 100 125   0
```

按 ASCII 读取并去掉末尾的空字节，得到：

```text
hgame{y0ur_m@th_1s_gO0d}
```

常量与输出还参考了 [oacia 的 HGAME2023 Reverse 题解](https://oacia.dev/hgame2023-reverse-writeup/) 交叉核对，但矩阵关系、完整脚本和验证条件均已写在正文中，无需依赖外链理解。

## 方法总结

本题主要考查把反编译循环还原成线性代数表达式。遇到 `savedregs` 一类反编译器生成的变量名时，应根据真实栈偏移还原指针指向；识别出矩阵乘法后，既可求逆，也可把 25 个字符作为线性方程未知数求解。无论采用哪种方法，都应把恢复结果重新代入原等式验证。
