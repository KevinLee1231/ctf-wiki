# week1_re2

## 题目简述

题目是 64 位 PE 自定义哈希逆向。程序要求输入 42 字符，格式为 `flag{...}`，取中间 36 字符经过带反馈的乘加、异或链式计算，生成 36 个 `uint32`，再与硬编码期望值比较。

算法不适合直接手工求逆，但每个字符范围和目标值都明确，可以用 Z3 建模求解。

## 解题过程

### 关键观察

主流程：

```text
strlen(input) == 42
input[:5] == "flag{"
input[-1] == "}"
hash(input[5:41]) == expected
```

哈希函数以 `DAT_140005040` 中的 36 个 `uint32` 为初始缓冲区，并按 5 组、每组 7 字符更新。核心形态为：

```text
new_val = ((c_next + 30) * c_cur - old_val) ^ prev_val
```

其中 `prev_val` 是上一轮计算结果，形成链式反馈。

### 求解步骤

用 36 个 8 bit 变量表示 flag 中间部分，限制为可打印 ASCII，并按反编译逻辑正向模拟哈希，最后约束每个输出等于硬编码目标。

```python
from z3 import *

chars = [BitVec(f"c{i}", 8) for i in range(36)]
s = Solver()
for ch in chars:
    s.add(ch >= 0x20, ch <= 0x7e)

# 按反编译代码模拟 hash_buffer 更新，并添加 hash_buffer[i] == expected[i]
# 这里省略常量表展开，实际脚本保留完整 expected 和初始 buffer。

assert s.check() == sat
m = s.model()
mid = bytes(m[ch].as_long() for ch in chars).decode()
print(f"flag{{{mid}}}")
```

求解结果：

```text
flag{159d1b01-b34e-44b9-b6fc-28438d6ede90}
```

## 方法总结

- 核心技巧：把不可直接逆的自定义哈希正向建模成 SMT 约束。
- 识别信号：输入长度、字符范围、目标数组固定，且每轮运算只涉及有限宽整数时，Z3 往往比人工逆推更稳。
- 复用要点：建模时要保持整数位宽和溢出语义；C 中 `uint32` 运算不能直接用无限精度整数替代。
