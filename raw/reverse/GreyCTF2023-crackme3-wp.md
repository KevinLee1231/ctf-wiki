# GreyCTF 2023 Crackme3

## 题目简述

题目实现了一台 64 寄存器的自定义虚拟机。每条指令为 32 位，包含操作码、目标寄存器、源寄存器和立即数；基础操作包括输入输出、位运算、加减移位及条件前跳。程序还会用 RC4 风格的交换过程持续重排操作码、输出操作和寄存器映射，`UNLOCK` 指令出现后连指令中的扰动位也开始触发表的自修改。

## 解题过程

生成器表明输入固定为 24 个可打印字符。它先打乱输入在虚拟寄存器中的位置，再把相邻位置组成一条链。每组约束取两个字符，经过加法、常量噪声、某个加法变体和低 8 位掩码后与固定值异或；任何一组不满足都会把 verdict 寄存器置为非零。

手工恢复全部动态映射很容易在某次交换后错位，因此直接对真实二进制做符号执行更稳。核心做法是建立 24 个符号字节和末尾换行，限制为可打印字符，让 angr 执行程序实际的解释器与自修改表，并以输出字符串筛选路径：

```python
import angr
import claripy

project = angr.Project("./crackme3", auto_load_libs=False)
chars = [claripy.BVS(f"c{i}", 8) for i in range(24)]
stdin = claripy.Concat(*chars, claripy.BVV(b"\n"))
state = project.factory.full_init_state(stdin=stdin)

for ch in chars:
    state.solver.add(ch >= 0x20, ch <= 0x7e)

simgr = project.factory.simulation_manager(state)
simgr.run()
for candidate in simgr.deadended:
    out = candidate.posix.dumps(1)
    if b"Correct password!" in out:
        print(candidate.solver.eval(claripy.Concat(*chars), cast_to=bytes))
```

这种方式让符号执行引擎沿实际代码维护四张置换表，不需要猜测 `UNLOCK` 之后的操作码含义。成功路径给出：

```text
grey{r_y0u_d1zzy?_9bfad}
```

再把该输入交回原程序，可看到 `Correct password!`，与生成器中的所有成对字符约束一致。

## 方法总结

本题的混淆建立在“指令语义会随执行历史变化”上，静态地给 0 到 63 号操作码贴标签并不可靠。可以选择完整复刻解释器，也可以让符号执行直接覆盖原解释器；无论哪种方法，都必须保留交换表的时序状态。输出成功分支比依赖某个易漂移的代码地址更适合作为求解条件。
