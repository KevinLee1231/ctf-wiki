# The Matrix

## 题目简述

程序把输入拆成若干 $3\times3$ 明文矩阵，依次用四个密钥矩阵左乘，并与七个静态密文矩阵比较。相邻明文窗口只前进 6 字节，所以每个 9 元素矩阵与下一块重叠 3 个元素。目标是恢复结构体布局、四个密钥的真实顺序，再求逆矩阵还原输入。

## 解题过程

### 1. 恢复矩阵结构和函数语义

矩阵对象在 64 位程序中的布局为：

```c
struct Matrix {
    unsigned char rows;       // offset 0
    unsigned char columns;    // offset 1
    unsigned char padding[6];
    unsigned int *elements;   // offset 8
};                            // sizeof = 0x10
```

反编译器无法自动识别该自定义结构，因此初始化代码表现为大量相似赋值。整理后可见程序调用四次初始化函数构造四个 $3\times3$ 密钥矩阵；必须按栈上的实际使用顺序排列为 `k2, k4, k1, k3`。

输入处理函数先检查长度，再把连续字符填入 $3\times3$ 明文矩阵。

核心运算函数实现普通矩阵乘法。对每一块有：

$$
B=KA
$$

因此：

$$
A=K^{-1}B
$$

### 2. 精确求逆并拼接重叠块

使用 SymPy 的有理数矩阵运算可以避免浮点逆矩阵带来的取整误差：

```python
from sympy import Matrix

k1 = [1, 3, 1, 0, 2, 1, 1, 2, 0]
k2 = [1, 2, 2, 3, 5, 6, 0, 2, 1]
k3 = [0, 4, 3, 1, 2, 1, 2, 3, 1]
k4 = [1, 2, 3, 3, 5, 6, 1, 4, 10]

ciphertext = [
    0x1E8, 0x1C0, 0x181, 0x557, 0x4D3, 0x41E, 0x13D, 0x111, 0x102,
    0x26C, 0x140, 0x145, 0x5B7, 0x2EC, 0x2F3, 0x5E9, 0x31D, 0x336,
    0x14D, 0x10A, 0x192, 0x0BD, 0x09F, 0x0F5, 0x0BD, 0x0A1, 0x101,
    0x162, 0x147, 0x223, 0x0FB, 0x0C0, 0x126, 0x191, 0x123, 0x1B7,
    0x0F0, 0x0FD, 0x10D, 0x29E, 0x2C0, 0x2F1, 0x091, 0x09F, 0x0A4,
    0x229, 0x13B, 0x12E, 0x4E4, 0x2D8, 0x2C7, 0x5BD, 0x325, 0x2E4,
    0x1C7, 0x151, 0x0D5, 0x0FE, 0x0E5, 0x06E, 0x12C, 0x0A0, 0x09E,
]

keys = [Matrix(3, 3, values) for values in (k2, k4, k1, k3)]
blocks = [
    Matrix(3, 3, ciphertext[offset:offset + 9])
    for offset in range(0, len(ciphertext), 9)
]

recovered = []
for index, block in enumerate(blocks):
    plain = keys[index % 4].inv() * block
    assert all(value.is_Integer for value in plain)
    values = [int(value) for value in plain]

    # 每个后继窗口与前一窗口重叠 3 字节，故通常只追加前 6 个。
    if index + 1 < len(blocks):
        recovered.extend(values[:6])
    else:
        recovered.extend(values)

print(bytes(recovered).rstrip(b"\x00").decode())
```

输出为：

```text
0xGame{78d51c59-6dc3-30d2-1276-18e13f80c478}
```

## 方法总结

矩阵逆向题通常有两层工作：先从指针访问和内存对齐恢复数据结构，再把程序循环翻译成线性代数。这里除了求 $K^{-1}B$，还必须处理密钥实际顺序以及 9 字节窗口每次只前进 6 字节的重叠关系；使用精确有理数运算可避免“浮点后手工四舍五入”带来的不确定性。
