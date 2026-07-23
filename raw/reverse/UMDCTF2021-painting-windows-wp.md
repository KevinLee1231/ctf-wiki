# Painting Windows

## 题目简述

题目给出 Windows PE。程序先调用 `IsDebuggerPresent` 做基础反调试，再逐字节变换输入并与静态数组比较。变换为先与 `0x0f` 异或，再乘 2。

## 解题过程

静态分析校验循环可还原为：

```c
target[i] == (input[i] ^ 0x0f) * 2;
```

乘 2 在题目目标字节范围内没有发生需要处理的模回绕，因此逆变换为：

$$
\text{input}[i]=\left(\frac{\text{target}[i]}{2}\right)\oplus 0x0f.
$$

从二进制中导出目标数组，逐字节还原：

```python
target = bytes.fromhex(
    "b4849698b69244e8ac7eb4a0b8f6dcfaf67896a0ec80f4ba"
    "a0b07cc2d67ef0b8a08a7eb4ba82d4ace4"
)
flag = bytes((value // 2) ^ 0x0F for value in target)
print(flag.decode())
```

得到：

```text
UMDCTF-{Y0U_Start3D_yOuR_W1nd0wS_J0URNeY}
```

`IsDebuggerPresent` 只影响动态跟踪：可以静态跳过，也可以在调试器中修改返回值；它不参与字符算法。

## 方法总结

简单反调试与核心校验应分开分析。找到比较点后向前做数据流切片，就能把每字节变换写成代数式并直接求逆。对于乘法逆变换，还要检查是否存在溢出或模 $256$ 语义；本题目标值均可直接除以 2。
