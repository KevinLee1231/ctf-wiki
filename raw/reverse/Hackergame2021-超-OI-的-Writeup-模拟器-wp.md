# 超 OI 的 Writeup 模拟器

## 题目简述

服务端为每个编号动态生成一个 ELF 校验程序，三级难度逐步加入 Function Merging、Control Flow Flattening 和 Mixed Boolean Arithmetic（MBA）混淆。核心校验是若干轮类似 Feistel 的 128 位变换；最终等级要求自动处理共 256 个样本。前两级用于暴露未充分混淆时的结构，真正目标是写出不依赖固定地址的自动分析器。

## 解题过程

### 为什么 Feistel 可以逆

把两半记为 $(L_i,R_i)$。典型一轮为：

$$
L_{i+1}=R_i,\qquad R_{i+1}=L_i\oplus F(R_i)
$$

反向计算只需要再次求值轮函数：

$$
R_i=L_{i+1},\qquad L_i=R_{i+1}\oplus F(L_{i+1})
$$

因此无需证明 $F$ 可逆，只要能把混淆后的轮函数当黑盒执行即可。

### 自动定位关键块

对每个下载的 ELF 执行以下流程：

1. 用 Binary Ninja 或其他 IR 框架找到 `main` 里的校验调用；
2. 进入被调用函数，在 SSA/HLIL 中找递归调用自身的基本块；
3. 含最终不等比较的块是目标常量比较，另一个是输入变换；
4. 记录轮函数调用点、左右半部变量和最多 16 轮的回边。

选择中层或高层 IR 的目的不是依赖变量名，而是把寄存器、栈槽与扁平化控制流统一成较稳定的数据流。

### 黑盒执行与 MBA 合成

用 Qiling 运行 ELF 到 `main` 之前，完成装载器和 libc 初始化；随后把 PC 设置到关键调用点，向参数寄存器写入随机值，直接收集输入输出对。对于题目限定的 MBA 运算集合：

```text
AND, OR, XOR, MUL, ADD, SUB
```

可以给每种运算编号，用 Z3 从四组随机样本中选出唯一符合关系的运算：

```python
import z3

def choose(op, a, b):
    return z3.If(op == 0, a & b,
           z3.If(op == 1, a | b,
           z3.If(op == 2, a ^ b,
           z3.If(op == 3, a * b,
           z3.If(op == 4, a + b, a - b)))))

solver = z3.Solver()
op = z3.BitVec("op", 3)
for a, b, observed in samples:
    solver.add(choose(op, z3.BitVecVal(a, 64),
                         z3.BitVecVal(b, 64)) == observed)
solver.add(z3.ULE(op, 5))
assert solver.check() == z3.sat
```

同样的方法可从比较块恢复变换后的目标左右半部。之后按 Feistel 逆公式逐轮回推：轮函数中的复杂表达式继续交给 Qiling 求值，左右半部的组合运算用已识别的 MBA 运算反解。

### 验证与批处理

每回推一轮，将两个 64 位值按小端拼成 16 字节候选；只保留可打印 ASCII，并用新的 Qiling 实例把候选喂给原程序。看到 `Correct code` 才接受，避免因轮数变化或错误 IR 匹配产生假阳性。

自动化主循环可概括为：

```python
for challenge_id in range(256):
    binary = download(challenge_id)
    analysis = locate_blocks(binary)
    target = solve_comparison_mba(analysis)

    for candidate in invert_up_to_16_rounds(analysis, target):
        if is_printable(candidate) and emulate_check(binary, candidate):
            submit(challenge_id, candidate)
            break
    else:
        raise RuntimeError(f"no valid input for {challenge_id}")
```

这里的函数名表示处理阶段，不是可直接运行的完整库接口。真正实现时应缓存分析结果、为 IR 模式写断言，并在某个样本结构不匹配时停止，而不是带着错误状态继续提交。

## 方法总结

- 核心技巧：在 IR 中自动定位校验数据流，用模拟器把复杂轮函数当黑盒，用少量样本和 Z3 识别 MBA 运算，再按 Feistel 结构反推输入。
- 识别信号：大量动态生成的同构二进制、控制流扁平化、混合布尔算术，以及左右半部交换的轮结构。
- 复用要点：自动化逆向应依赖结构特征而非固定地址；每个候选都必须回到原二进制动态验证，IR 和反编译器的分析错误也要显式处理。
