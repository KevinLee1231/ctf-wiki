# GreyCTF2022 - MBA

## 题目简述

程序用大量 mixed Boolean-arithmetic 表达式混淆一个 8 字节口令的哈希，并把结果与 `0x3dd99b6c9d29c576` 比较。混淆层很长，但所有运算都在固定宽度位向量上，适合按源码原样建模给 SMT 求解器。

## 解题过程

先确定哈希状态初值 `0x5ca1ab1ef01dab1e`、每轮输入字节位置和 64 位回绕语义。不要把 C 的无符号溢出翻成 Python 无限精度整数；使用 Z3 的 `BitVec(64)` 可自动保留模 $2^{64}$ 运算。

```python
from z3 import *

chars = [BitVec(f'c{i}', 8) for i in range(8)]
state = BitVecVal(0x5ca1ab1ef01dab1e, 64)
for c in chars:
    state = mba_round(state, ZeroExt(56, c))

s = Solver()
s.add(state == 0x3dd99b6c9d29c576)
for c in chars:
    s.add(c >= 0x20, c < 0x7f)
assert s.check() == sat
password = bytes(s.model()[c].as_long() for c in chars)
```

将恢复的口令提交给服务，后续管理员消息认证通过并返回：

```text
grey{A_M4st3r_B1n4ry_An4Lyst_OOOO}
```

## 方法总结

MBA 混淆的目的通常是隐藏简单的位运算等价式。可以先用恒等式化简，也可以忠实构造固定位宽 SMT 模型；无论哪种方式，都必须还原整数宽度、符号扩展和移位语义。求得口令后应运行原程序确认哈希完全一致。
