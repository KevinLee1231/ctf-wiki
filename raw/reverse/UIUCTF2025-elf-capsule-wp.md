# ELF Capsule

## 题目简述

附件是运行在 QEMU `virt` 机器上的 RISC-V 64 位 ELF。外层 `main.elf` 运行于 S-mode，内部嵌入的 child ELF 运行于 U-mode，二者共享地址空间。child 通过故意访问未映射的 `0x0..0x1000` 触发异常；S-mode handler 根据错误地址把这些 fault 解释成栈式计算器、输入输出和停止操作，相当于一套“memory-mapped syscall”。目标是还原这套虚拟运算并用 Z3 求 24 个未知可打印字符。

## 解题过程

输入总长固定为 32 字节：已知前缀 `uiuctf{`、24 字节未知正文和右花括号。child 逐字节读取输入并在 8 位模 $2^8$ 意义下取负数。令结果为 $v_i=-x_i\bmod 256$，程序构造四个 32 字节区：

$$
\begin{aligned}
G_1[i]&=v_i\oplus 0x29,\\
G_2[i]&=v_i+0xae\pmod{256},\\
G_3[i]&=v_i\oplus(G_2[i]-G_1[i])\pmod{256},\\
G_4[i]&=(v_i\ll4)\mathbin{|}(v_i\gg4).
\end{aligned}
$$

128 字节内存顺序不是简单的四段顺排，而是：

```text
G1 || reverse(G2) || G3 || reverse(G4)
```

随后按小端序打包成 16 个 64 位栈元素。第一段归约对 $i=1..7$ 执行：

$$
u_{15-i}\leftarrow u_{15-i}+
\left(\operatorname{ROL}_{64}(u_{16-i},i^2)\oplus C\right),
$$

其中 $C=0x9e3779b97f4a7c15$，并要求 `u64s[8] == 4034066512636806762`。第二段令 $j=8..14$，执行旋转、异或后再减去下一项，并分别检查：

```text
u64s[2] == 546867345586458711
u64s[0] == 6860906515746073210
```

把上述运算逐字翻译为 8 位和 64 位 `BitVec` 即可：

```python
import z3

xs = [z3.BitVec(f"x{i}", 8) for i in range(24)]
s = z3.Solver()
for x in xs:
    s.add(x >= 32, x <= 126)

all_bytes = [z3.BitVecVal(x, 8) for x in b"uiuctf{"] \
          + xs \
          + [z3.BitVecVal(ord("}"), 8)]
v = [-x for x in all_bytes]
g1 = [x ^ 0x29 for x in v]
g2 = [x + 0xae for x in v]
g3 = [v[i] ^ (g2[i] - g1[i]) for i in range(32)]
g4 = [(x << 4) | z3.LShR(x, 4) for x in v]
memory = g1 + list(reversed(g2)) + g3 + list(reversed(g4))
u64s = [z3.Concat(*reversed(memory[i:i + 8])) for i in range(0, 128, 8)]

C = z3.BitVecVal(0x9e3779b97f4a7c15, 64)
for i in range(1, 8):
    u64s[15 - i] = u64s[15 - i] + (z3.RotateLeft(u64s[16 - i], i * i) ^ C)

for j in range(8, 15):
    u64s[14 - j] = (
        z3.RotateLeft(u64s[15 - j], j * j) ^ C
    ) - u64s[14 - j]

s.add(u64s[8] == z3.BitVecVal(4034066512636806762, 64))
s.add(u64s[2] == z3.BitVecVal(546867345586458711, 64))
s.add(u64s[0] == z3.BitVecVal(6860906515746073210, 64))

assert s.check() == z3.sat
m = s.model()
print(bytes(m.eval(x).as_long() for x in xs))
```

无额外字符约束的完整枚举可能较慢；官方模型规定应在 10 分钟内完成。本地对仓库候选值重新注入同一组约束，Z3 返回 SAT，24 字节正文为：

```text
M3m0Ry_M4ppED_SysTEmca11
```

最终 flag：

```text
uiuctf{M3m0Ry_M4ppED_SysTEmca11}
```

## 方法总结

- 核心技巧：把异常地址当成自定义 syscall 编号，恢复 child ELF 的栈式虚拟机，再用定宽 BitVec 精确模拟溢出、旋转和小端打包。
- 识别信号：U-mode 程序频繁访问固定的低地址并由 S-mode handler 继续执行，说明 fault 本身是预期控制流而不是崩溃。
- 复用要点：SMT 建模要保留 8/64 位截断、逻辑右移和端序；使用 Python 无界整数或普通右移会悄悄改变约束含义。
