# ZzZ

## 题目简述

附件是 Visual Studio 2022 生成、未附 PDB 的 Debug PE。通过提示字符串 `enter flag` 的交叉引用可定位主校验函数。UUID 首段和末段是常量，中间三个 4 字符分组被按小端序解释为 32 位无符号整数，再进入三条含溢出和位运算的方程。

## 解题过程

在 IDA Strings 窗口找到输入提示，跟随交叉引用到主函数。反编译后可提取格式和常量：

```text
format = 0xGame{%8x-%4s-%4s-%4s-%12x}
first  = 3846448765 = 0xe544267d
last   = 0xd085a85201a4
```

三个 `%4s` 会各写入连续 4 字节，但程序随后把这 4 字节当作 `unsigned int`。在 x86 小端序下，字符串 `7812` 对应整数 `0x32313837`，因此 Z3 解出的整数必须用 `to_bytes(4, "little")` 转回字符。

方程在 32 位整数上运算，使用 `BitVec(32)` 才能正确保留模 $2^{32}$ 溢出和位移语义：

```python
from z3 import BitVecs, Solver, sat

v10, v11, v12 = BitVecs("v10 v11 v12", 32)
solver = Solver()

solver.add(11 * v11 + 14 * v10 - v12 == 0x48FB41DDD)
solver.add(9 * v10 - 3 * v11 + 4 * v12 == 0x2BA692AD7)
solver.add(((v12 - v11) >> 1) + (v10 ^ 0x87654321) == 3451779756)

assert solver.check() == sat
model = solver.model()

parts = [
    model[var].as_long().to_bytes(4, "little").decode()
    for var in (v10, v11, v12)
]

flag = "0xGame{%08x-%s-%s-%s-%012x}" % (
    3846448765,
    parts[0],
    parts[1],
    parts[2],
    0xD085A85201A4,
)
print(parts)
print(flag)
```

输出：

```text
['7812', '44b3', 'a35d']
0xGame{e544267d-7812-44b3-a35d-d085a85201a4}
```

## 方法总结

本题的难点是从无符号 PE 中定位校验函数，以及同时处理 32 位溢出、算术右移和小端序字符解释。用整数方程而不是 BitVec 会丢失溢出语义；解出 BitVec 后直接按十进制打印也会丢失原始字符顺序。
