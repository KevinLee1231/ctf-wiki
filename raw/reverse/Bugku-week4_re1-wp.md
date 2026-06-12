# week4_re1

## 题目简述

题目是简单 VM 逆向。64 位 Windows PE 读取 42 字符输入，执行 `DAT_00404020` 中的字节码。字节码操作输入、寄存器数组和输出区，最后比较 `DAT_0040c060[100..141]` 与 `DAT_00408020`。

VM 指令模式固定，每个输入字符独立变换，因此可从字节码归纳出通项公式并直接反解。

## 解题过程

### 关键观察

VM 指令每条由 3 个 `int32` 组成，操作码如下：

```text
1: R[a] ^= R[b]
2: R[a] += R[b]
3: R[a] -= R[b]
4: R[a] <<= R[b] & 0x1f
5: R[a] >>= R[b] & 0x1f
6: OUT[a] = R[b]
7: R[a] = IN[b]
8: R[a] = imm
0: halt
```

对每个字符，字节码模式等价于：

```text
OUT[i] = ((IN[i] ^ (32 + i)) << (i % 3)) + (33 + i)
```

### 求解步骤

从 `DAT_00408020` 提取 42 个 `uint32` 期望值后按公式反解：

```python
expected = [...]  # 42 uint32 from DAT_00408020
flag = []
for i, y in enumerate(expected):
    xor_key = 32 + i
    shift = i % 3
    x = ((y - (33 + i)) >> shift) ^ xor_key
    flag.append(x)

print(bytes(flag).decode())
```

输出：

```text
flag{af3fd248-41b4-4884-9f6b-747878be8e74}
```

## 方法总结

- 核心技巧：把 VM 字节码归纳为每字符独立公式，而不是完整写解释器。
- 识别信号：固定长度指令、寄存器数组、输入/输出数组和 opcode switch 组合出现时，应先建立指令语义表。
- 复用要点：如果字节码模式重复，直接抽象通项公式可以大幅简化；但抽象后仍要用期望数组正向验证。
