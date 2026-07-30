# L3akCTF 2024 Angry Writeup

## 题目简述

程序读取一行密码，并通过 `compare` 与 `more_checks` 中的大量逐字节算术、异或和长度条件判断是否正确。没有加密数据或动态解密层，决定性任务是把分散的约束转成 SMT 方程。

`get_len()` 返回：

$$
\frac{222}{2+2+2}=37,
$$

所以输入恰有 37 个字符。`more_checks` 又直接固定了下标 8、9、23、29，能显著减少多解。

## 解题过程

为每个字符建立一个 8 位 BitVec，并限制为可打印 ASCII：

```python
from z3 import *

xs = [BitVec(f"x_{i}", 8) for i in range(37)]
s = Solver()

for x in xs:
    s.add(x >= 0x20, x <= 0x7e)

s.add(xs[0] == ord("L"))
s.add(xs[1] == ord("3"))
s.add(xs[2] == ord("A"))
s.add(xs[3] == ord("K"))
s.add(xs[4] == ord("{"))
s.add(xs[36] == ord("}"))
```

随后逐项翻译判断。例如：

```c
input[5] * input[15] - input[7] == 4844
-input[7] * -input[7] == 10609
input[13] ^ input[0] ^ input[1] ^ input[2] ^ input[3] == 68
input[14] + input[15] + input[16] + input[17] == 348
```

对应：

```python
s.add(xs[5] * xs[15] - xs[7] == 4844)
s.add((-xs[7]) * (-xs[7]) == 10609)
s.add(xs[13] ^ xs[0] ^ xs[1] ^ xs[2] ^ xs[3] == 68)
s.add(xs[14] + xs[15] + xs[16] + xs[17] == 348)
```

把第二层检查也加入：

```python
s.add(xs[8] == 0x72)
s.add(xs[9] == 0x5f)
s.add(xs[23] == (444 >> 2))
s.add(xs[29] == 0x34)
s.add((xs[10] & ~0x30) < 10)
```

源码中有几处未加括号的 `^` 与 `==` 混用。C 的相等比较优先级高于按位异或，而 Python/Z3 表达式的解析顺序不同，不能直接复制原文本后假定语义一致。最可靠的方法是对照反汇编中的实际分支，或给每个预期子式显式加括号。即使这些弱条件按编译后的真实语义建模，`more_checks` 和其他方程仍足以确定官方输入。

求一个满足模型并拼接字符：

```python
assert s.check() == sat
m = s.model()
answer = "".join(chr(m.eval(x).as_long()) for x in xs)
print(answer)
```

得到：

```text
L3AK{angr_4_l1f3_d0nt_do_iT_m4nU4lly}
```

## 方法总结

- 对固定长度、逐字节运算的 checker，Z3 比手工回代更稳妥；也可以用 angr 直接寻找成功分支。
- 建模前要确定字节宽度、符号扩展和运算符优先级。这里用 8 位 BitVec 与可打印范围，既贴近 `char` 输入，也避免无意义的控制字符解。
- `malloc(sizeof(char))` 后读取 40 字节确实存在堆溢出，但程序没有要求利用它；恢复满足条件的输入才是决定性主障碍，因此仍归入 Reverse。
