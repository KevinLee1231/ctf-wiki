# rbf

## 题目简述

题目是一个 Android/Java 与 Rust JNI 混合程序。Java 层只检查输入格式并调用 `libmyrust` 中的本地校验函数；本地库在初始化阶段用 RC4 解密一段约 32 KiB 的代码，随后用 Brainfuck 解释器执行它。解密后的 Brainfuck 程序最终建立关于 flag 内部 12 个字符的线性约束。

虽然载体包含 Android 和 JNI，但决定性障碍是恢复本地库、解密代码和自定义解释执行逻辑，因此归类为 Reverse。

## 解题过程

### Java 层约束与 JNI 入口

Java 层先做固定格式检查：

```java
input.length() == 22
input.startsWith("miniLCTF{")
input.endsWith("}")
```

格式部分共占 10 个字符，所以花括号内恰好有 12 个待求字符。通过 JNI 导出名可以定位到本地入口：

```text
Java_com_doctor3_rbf_MainActivity_Check
```

本地库名为 `libmyrust`。反编译入口附近可以看到按 `+`、`,`、`-` 等字符分派的大型 `switch`，这些正是 Brainfuck 的指令字符，说明本地代码实现了一个 Brainfuck 解释器，而不是直接校验输入。

### 解密 Brainfuck 程序

查看 `.init_array` 可定位到初始化函数 `sub_23820`。该函数包含 RC4 的密钥调度和伪随机流生成过程，使用的 ASCII 密钥是：

```text
myrust
```

程序申请并复制 `0x7D1E` 字节数据，即十进制 32030 字节，再逐字节与 RC4 密钥流异或。RC4 加解密使用同一操作，因此恢复代码的流程是：

```python
def rc4(data: bytes, key: bytes) -> bytes:
    state = list(range(256))
    j = 0
    for i in range(256):
        j = (j + state[i] + key[i % len(key)]) & 0xFF
        state[i], state[j] = state[j], state[i]

    i = j = 0
    out = bytearray()
    for value in data:
        i = (i + 1) & 0xFF
        j = (j + state[i]) & 0xFF
        state[i], state[j] = state[j], state[i]
        stream = state[(state[i] + state[j]) & 0xFF]
        out.append(value ^ stream)
    return bytes(out)

```

取得加密数据后，调用方式为 `bf_code = rc4(encrypted_blob, b"myrust")`。现存 PDF 没有保留这 32030 字节密文附件，所以无法在当前材料上直接执行这一步；算法、密钥和长度均来自反编译证据。

### 从 Brainfuck 宏提取线性约束

解密结果是一段很长的 Brainfuck 程序，但其中包含大量重复片段。题解识别出的两个关键宏为：

```text
eq : [->-<]+>[<->[-]]
mul: >>[-]>[-]<<<[>>>+<<<-]>>>
```

可以先把这些固定片段替换为语义标记，再按重复的比较块切分程序。每一块本质上都在对输入字符的偏移量做乘加并与常数比较。令花括号内字符为 $c_i$，并定义

$$
x_i=\operatorname{ord}(c_i)-\operatorname{ord}('a'),
\qquad 0\le x_i<26,
$$

即可把 Brainfuck 校验改写成 Z3 的整数线性约束。求解框架如下：

```python
from z3 import Int, Solver

x = [Int(f"x{i}") for i in range(12)]
solver = Solver()
for value in x:
    solver.add(0 <= value, value < 26)

def add_equation(coefficients, constant):
    expression = sum(a * value for a, value in zip(coefficients, x))
    solver.add(expression == constant)
```

对每个比较块调用一次 `add_equation`；完整加入 12 个方程后，再执行 `solver.check()` 并按 `chr(model[x[i]] + ord('a'))` 还原字符。

PDF 第 4 页的方程右侧有裁切，不能把不完整的行伪装成可运行脚本。页面中仍有两条完整可读的约束：

$$
2x_2+3x_6+2x_7+3x_8+2x_{10}=147,
$$

$$
3x_1+3x_5+2x_6+2x_{10}+x_{11}=102.
$$

页面给出的模型对应内部字符串 `favyxwekppoa`，即

```text
[5, 0, 21, 24, 23, 22, 4, 10, 15, 15, 14, 0]
```

代入上面两条完整方程分别得到 147 和 102，与页面常数一致；字符串长度也是 12，拼回外层格式后总长度为 22。

最终结果为：

```text
miniLCTF{favyxwekppoa}
```

### 证据边界

现存材料只有四页 PDF，没有 APK、本地库、加密数据或完整 Brainfuck 文本。PDF 中的第 4 页还裁掉了部分方程右端。因此能够可靠确认 Java 格式检查、JNI 入口、Brainfuck 虚拟机、RC4 密钥与数据长度、宏化简方法、两条完整方程以及公布的模型；无法从现有仓库独立重放 RC4 解密或重新求解全部约束。本文没有补写被裁切的方程。

## 方法总结

这道题需要逐层剥离载体：Java 层确定输入形状，JNI 层暴露解释器，初始化函数揭示 RC4，解密后的 Brainfuck 再化简为线性方程。面对数万字节的重复 Brainfuck，直接逐指令阅读效率很低；识别固定宏、切分重复块并转为约束求解，才是稳定的分析路径。

最终 flag 为：

```text
miniLCTF{favyxwekppoa}
```
