# UMDCTF 2019 - Armageddon

## 题目简述

附件是 32 位 ARM ELF。主函数要求输入 41 字节字符串，并依次调用 41 个小型校验函数；每个函数只约束若干字符之间的算术或位运算关系。手工逐式求解成本较高，适合用符号执行汇总约束。

## 解题过程

反汇编主函数后，先收集顺序调用的 41 个校验函数地址。每个函数都接收同一个字符串指针，返回到调用者即表示当前约束成立。可以分别从函数入口建立 `call_state`，把 41 个可打印符号字节写入固定内存，并让 angr 探索到人为设置的返回地址：

```python
import angr
import claripy

project = angr.Project("./armageddon", auto_load_libs=False)
functions = [
    0x104F4, 0x106D8, 0x1082C, 0x10A10, 0x10BD0,
    0x10D54, 0x10F08, 0x11070, 0x111F4, 0x1136C,
    0x11504, 0x1170C, 0x118C0, 0x11A28, 0x11BE8,
    0x11D9C, 0x11F74, 0x12128, 0x12348, 0x1252C,
    0x126B0, 0x12864, 0x129A8, 0x12B80, 0x12D94,
    0x12EFC, 0x130D4, 0x132A8, 0x13440, 0x13654,
    0x13838, 0x139D0, 0x13BB4, 0x13D1C, 0x13ED0,
    0x14078, 0x1422C, 0x143B0, 0x14528, 0x146AC,
    0x148BC,
]

chars = [claripy.BVS(f"c{i}", 8) for i in range(41)]
buffer = claripy.Concat(*chars, claripy.BVV(0, 8))
constraints = []

for address in functions:
    state = project.factory.call_state(
        address,
        0x300000,
        ret_addr=0x41414140,
        add_options={
            angr.options.ZERO_FILL_UNCONSTRAINED_MEMORY,
            angr.options.ZERO_FILL_UNCONSTRAINED_REGISTERS,
        },
    )
    state.memory.store(0x300000, buffer)
    for char in chars:
        state.solver.add(char >= 0x20, char <= 0x7e)

    manager = project.factory.simulation_manager(state)
    manager.explore(find=0x41414140)
    solved = manager.found[0]
    constraints.extend(
        item for item in solved.solver.constraints
        if any(name.startswith("c") for name in item.variables)
    )

solver = claripy.Solver()
solver.add(constraints)
for index, value in enumerate(b"UMDCTF-{"):
    solver.add(chars[index] == value)
solver.add(chars[-1] == ord("}"))
print(solver.eval(claripy.Concat(*chars), 1)[0].to_bytes(41, "big"))
```

加入标准 flag 前缀、后缀和可打印范围后，约束组给出唯一结果：

```text
UMDCTF-{ARM_1s_s0_SATisfying_7y8fdlsjebn}
```

SHA-256 与官方摘要一致。

## 方法总结

这题的难点是大量分散的关系约束，而不是某个单独的复杂算法。把每个校验函数独立执行，可以避免整程序路径爆炸；再将成功路径上与输入有关的约束合并，求解会稳定得多。符号执行结果仍应通过长度、固定格式和官方摘要交叉验证。
