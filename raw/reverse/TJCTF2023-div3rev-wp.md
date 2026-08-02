# div3rev

## 题目简述

Python 检查器把 27 字节输入递归三等分，每一层分别对左、中、右分支应用 `op1`、`op2`、`op3`，最后与固定 27 字节结果比较。递归深度为 3，所以每个位置经历的操作序列由索引的三进制位决定。

## 解题过程

先化简三个操作：

- `op1` 中耗时 420 次的乘法结果没有被使用；其有效行为是“偶数字节加 8，奇数字节不变”，逆操作为对偶数减 8；
- `op2` 连续 XOR 69 共 100 次，偶数次 XOR 相互抵消，只剩加 12，逆操作为减 12；
- `op3` 把最低位移到最高位，即 8 位循环右移 1，逆操作为循环左移 1。

对索引 $i$，从递归外层到内层的分支分别为 $\lfloor i/9\rfloor$、$\lfloor(i\bmod9)/3\rfloor$、$i\bmod3$。解密时按相反顺序应用对应逆变换：

```python
def inv1(value):
    return value - 8 if value % 2 == 0 else value

def inv2(value):
    return value - 12

def inv3(value):
    return ((value << 1) & 0xff) | (value >> 7)

inverse = [inv1, inv2, inv3]
cipher = bytes.fromhex(
    "8c86b19086c93dbe9b8087ca868d4b4ac4653fb cdb43be215920af".replace(" ", "")
)

plain = bytearray(cipher)
for i in range(len(plain)):
    for branch in ((i % 27) // 9, (i % 9) // 3, i % 3):
        plain[i] = inverse[branch](plain[i])
print(plain.decode())
```

结果为：

```text
tjctf{randomfifteenmorelet}
```

## 方法总结

- 逆向递归分治变换时，可把元素路径编码成基数位；这里三等分自然对应三进制索引。
- 先删去无数据流影响的耗时循环，再化简偶数次 XOR 等恒等操作，可以显著降低复杂度。
- 复原时必须反转操作应用顺序，并把旋转限制在 8 位范围内。
