# SMCProMax

## 题目简述

程序运行时用 `0x90` 异或解密自身代码，静态反编译最初只能看到 SMC 解密器。解开并修补代码后，校验函数把输入分成十个 32 位 word，分别执行 32 轮条件移位；该约束可交给 Z3 求逆。程序在进入校验前还偷偷改了一个字符，必须反推用户实际应输入的值。

## 解题过程

先动态运行到 SMC 解密循环结束，此时目标区域已经逐字节异或 `0x90`。把解密后的内存 dump 出来，或直接在 IDA/x64dbg 中 patch 后重新分析，可得到如下 32 位变换：每轮按有符号数判断最高位，左移后在负数分支异或常量 `0xc4f3b4b3`。

```python
from z3 import BitVec, If, Solver, sat

targets = [
    2053092702, -490481854, -1704322843, -1418679088,
    86802781, 987171458, 965631658, 264545711,
    -1342106783, -825370173,
]

words = [BitVec(f"x{i}", 32) for i in range(len(targets))]

def transform(value):
    for _ in range(32):
        value = If(
            value < 0,
            (value << 1) ^ 0xC4F3B4B3,
            value << 1,
        )
    return value

solver = Solver()
for word, target in zip(words, targets):
    solver.add(transform(word) == target)

assert solver.check() == sat
model = solver.model()
checked = bytearray(
    b"".join(
        model.eval(word).as_long().to_bytes(4, "little")
        for word in words
    )
)
print(checked)  # 校验函数实际看到的是 moectf{y0u_mu5t_know_vvHAt_1s__SMC__n0w}

# 在校验前，程序会把用户输入中的这一字节异或 0x12。
position = checked.index(ord("H"))
checked[position] ^= 0x12  # 0x48 ^ 0x12 == 0x5a，即 H -> Z
print(checked.decode())
```

Z3 还原出的“校验时缓冲区”为：

```text
moectf{y0u_mu5t_know_vvHAt_1s__SMC__n0w}
```

但用户输入在校验前会被异或 `0x12`，要让校验时出现 `H`，输入中必须放 `Z`，所以最终答案是：

```text
moectf{y0u_mu5t_know_vvZAt_1s__SMC__n0w}
```

## 方法总结

SMC 题要区分“磁盘中的加密代码”“运行时解密后的校验逻辑”和“进入校验前被修改的输入”三层状态。Z3 只解决最后校验函数的逆像，若忽略前置字符变换，模型给出的字符串仍不是实际应提交的 flag。
