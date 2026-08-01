# LLIR

## 题目简述

附件是 LLVM IR 形式的 flag checker。它约束 37 个输入字节，包括固定前缀 `byuctf`、零基下标 6 的 `{`、末位 `}`，以及大量位置相等和线性算术关系。所有条件最终以一个大布尔表达式合并。

LLVM IR 只是载体，决定性工作是把确定性的字节约束翻译给求解器。

## 解题过程

可先把 IR 编译成便于动态观察的程序：

```bash
llvm-as 'checker?.ll' -o checker.bc
clang checker.bc -o checker
```

不过无需逐条模拟指令。为 37 个字符建立 8 位 BitVec，限制为可打印 ASCII，然后把 IR 中的比较逐项转写。例如：

```python
from z3 import BitVec, Solver, sat

flag = [BitVec(f"flag_{i}", 8) for i in range(37)]
s = Solver()
for ch in flag:
    s.add(ch >= 0x20, ch <= 0x7d)

for i, ch in enumerate(b"byuctf{"):
    s.add(flag[i] == ch)
s.add(flag[36] == ord("}"))

s.add(flag[4] == flag[14], flag[14] == flag[17])
s.add(flag[9] + flag[20] == flag[31] + 3)
s.add(flag[31] + 3 == flag[0])
s.add(flag[10] == flag[7] + 6)
# 其余关系按 checker 中的条件继续加入。
```

求解结果为唯一可打印字符串：

```text
byuctf{lL1r_not_str41ght_to_4sm_458d}
```

把模型输出重新送入编译后的 checker，程序打印 `You win!!`，说明约束翻译无误。

## 方法总结

- 核心技巧：把 LLVM IR 中的大型确定性比较式转为 8 位符号约束，用 Z3 一次求解。
- 识别信号：输入长度固定、分支只包含字符相等与加减乘关系时，约束求解比手工回溯更直接。
- 复用要点：使用与原程序一致的位宽和有符号/无符号语义；模型必须回放原 checker，防止漏抄或错抄条件。
