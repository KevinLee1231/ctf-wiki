# Satisfiability

## 题目简述

二进制用约 70 条加减乘等式约束 60 个 flag 字节。手工联立十分繁琐，但这些条件可以原样交给 SMT 求解器。目标是正确建模 8 位字符并读取满足模型。

## 解题过程

从反编译结果提取所有形如下面的约束：

```text
flag[45] - flag[47] + flag[42] - flag[41] + flag[29] == 154
flag[28] - flag[54] - flag[53] * flag[59] * flag[5] == -1378125
```

题目生成器把每个字符建模为 8 位 `BitVec`，所以复现时也使用相同位宽，避免 Python 无界整数与机器字节运算语义不一致：

```python
from z3 import BitVec, Solver, sat

flag = [BitVec(f"f{i}", 8) for i in range(60)]
s = Solver()

for expression in equations:
    s.add(eval(expression, {"flag": flag}))

assert s.check() == sat
model = s.model()
answer = "".join(chr(model[x].as_long()) for x in flag)
print(answer)
```

求解器返回的模型为：

```text
grey{i_learnt_all_about_SMT_solvers_today_z3_or_cvc5_is_god}
```

## 方法总结

大量离散代数约束适合交给 SMT，而不是手工消元。建模的关键是变量数量、位宽、有符号/无符号语义与原程序一致；约束抄错一项就可能导致 `unsat` 或错误模型。
