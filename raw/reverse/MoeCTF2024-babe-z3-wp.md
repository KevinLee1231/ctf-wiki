# babe-z3

## 题目简述

程序把 32 字节输入按小端序解释为四个 `uint64_t`，用九组位运算、算术和取模条件校验。额外陷阱是 `switch_0` 的高、低 32 位分别承载真条件与假条件：最终既要求整个 64 位值非零，又要求截断后的低 32 位 `fake` 为零。

## 解题过程

第一条假条件若成立，会在 `switch_0` 低 32 位设置 `0x10000000`；下一条真条件成立则设置高位 `0x100000000`。最终检查为：

```c
DWORD fake = (DWORD)switch_0;
if (switch_0 && switch_1 && switch_2 && switch_3 &&
    switch_4 && switch_5 && switch_6 && switch_7 &&
    switch_8 && !fake)
    success();
```

所以必须约束假表达式不等于诱饵常量，而其余九项全部等于目标值。使用 64 位 `BitVec` 会自动模拟 `uint64_t` 回绕；无符号取模应显式使用 `URem`：

```python
from z3 import BitVecs, Solver, URem, sat

a1, a2, a3, a4 = BitVecs("a1 a2 a3 a4", 64)
solver = Solver()

fake_expr = (
    a3 & a1 & (~(a4 + a2) | (a2 & a2) | (a4 & a1 & ~a4))
) ^ (a4 & a3)
solver.add(fake_expr != 0xCAFEBABEDEADC0DE)

solver.add(
    (a1 & a2 & (~(a1 | a2) | (a3 & a2) | (a4 & a1 & ~a4))
     | (a4 & a1 & a3))
    == 2316015897844654385
)
solver.add(
    (a2 ^ (a4 & ~a1 | a4 & ~a3 | (a1 + a3) & a3
           | (a2 + a4) & a2 & ~a1))
    == 8102257287365753684
)
solver.add(((a1 - a2) ^ (a3 - a4)) == 287668530830180307)
solver.add(((a4 + a2) ^ (a1 + a3)) == 865433324338348261)
solver.add((a1 ^ a2 ^ a3 ^ a4) == 145809558366915671)
solver.add((a1 & a2 & a3 & a4) == 2314885599605039392)
solver.add(
    (a1 + a2 - a3 + a4) * (a1 + a3 - a4 + a2)
    == 1840182356754417097
)
solver.add(URem(a1 + a2 + a3 + a4, 114514) == 21761)
solver.add(URem(a1 * a2 * a3 * a4, 1919810) == 827118)

assert solver.check() == sat
model = solver.model()
answer = b"".join(
    model.eval(word).as_long().to_bytes(8, "little")
    for word in (a1, a2, a3, a4)
)
print(answer)
```

模型输出：

```text
9c0525dcbadf4cbd9715067159453e74
```

最终提交：

```text
moectf{9c0525dcbadf4cbd9715067159453e74}
```

## 方法总结

本题的关键是理解截断：`switch_0` 的高位可以让 64 位条件为真，同时低 32 位仍为零。原稿 solver 漏掉了三项真实约束，并把假条件常量抄成了另一个值，虽然偶然仍可能得到答案，但不等价于程序；这里已按源码补全并实际求解验证。
