# mt_speed_run

## 题目简述

服务实现了一个修改版 MT19937。内部状态仍是 624 个 32 位整数，共 19968 个未知比特，但 temper 阶段的四个掩码/反馈常量和一个左移位数被随机化：

```python
def temper(state):
    y = state
    y ^= y >> 11
    y ^= (y << 7) & MAGIC_A
    y ^= (y << 15) & MAGIC_B
    y ^= (y << MAGIC_E) & MAGIC_D
    y ^= y >> 18
    return y
```

题目先输出连续 30000 次 `temper(state) >> 31` 的结果，即每次只泄露一个最高位，再公布 `MAGIC_A` 至 `MAGIC_E`。选手需要在 10 秒内恢复当前 624 个状态整数，并提交

```python
sha256(','.join(map(str, state)).encode()).hexdigest()
```

因此主障碍不是传统的“收集 624 个完整输出后逐个 untemper”，而是从大量截断到 1 位的输出中快速求解完整状态。

## 解题过程

### 将生成器符号化为线性方程

MT19937 的移位、异或、按常量掩码，以及 twist 中由最低位选择常量的操作，都可以表示成 $\mathrm{GF}(2)$ 上的线性变换。把初始状态的 19968 个比特视为变量后，每个输出最高位都是这些变量的一个异或线性组合：

$$
o_i=\langle a_i,s\rangle\pmod 2.
$$

题目给出 30000 个 $o_i$，方程数多于未知数。只要按题目公布的五个 MAGIC 参数符号执行相同的 temper 和状态更新，就能构造一个过定二元线性方程组并恢复唯一状态。

官方 `mtsolver.cpp` 使用 `BitVec<32, 19968>` 表示每个符号状态字，依次加入约束：

```cpp
state.cursor = 0;
for (size_t i = 0; i < 30000; i++) {
    solver.add(state.next_trunc<1>(), observed_bits[i]);
}
solver.solve();
```

其中 `next_trunc<1>()` 完整模拟题目的 twist 与 temper，但只取最高一位建立方程。底层用 M4RI 对稠密 $\mathrm{GF}(2)$ 矩阵做消元，避免 Python 位集合或通用 SAT 求解器带来的时间开销。

### 为什么现成预计算不能直接套用

[MT19937 Symbolic Execution and Solver 的示例](https://github.com/JuliaPoo/MT19937-Symbolic-Execution-and-Solver/blob/master/Demo%20of%20Features.ipynb)展示了标准 Python MT19937 的符号执行：例如收集 `getrandbits(4)` 的截断输出，利用预计算矩阵可以在约 1 至 2 秒内克隆状态；不使用预计算时，也可以把输出位写成异或方程再求解。

该外链真正适用于本题的结论是“截断输出仍提供关于 19968 个状态位的线性约束”。它的预计算矩阵依赖标准 MT19937 的固定 temper/twist 常量，而本题随机生成 `MAGIC_A` 至 `MAGIC_E`，所以矩阵系数每次连接都会改变，必须现场重新构造。题目还限制 `MAGIC_A` 至 `MAGIC_D` 的汉明重量在 10 到 22 之间，避免掩码过于稀疏或稠密造成明显退化。

### 恢复并提交当前状态

服务在生成输出前先执行一次 `update_mt(state)`，随后每消费完 624 个输出再次更新。30000 个输出等于 48 个完整块再加 48 个输出；需要让符号执行的游标与服务端完全一致，最终求出的数组才是服务端计算摘要时的“当前状态”。

编译官方求解器需要 M4RI 与 fmt：

```bash
g++ -std=c++17 -O3 -march=native -o mtsolver mtsolver.cpp \
  $(pkg-config --cflags --libs m4ri fmt)
```

客户端解码 Base64 得到恰好 30000 个比特，写入求解器输入，同时写入五个 MAGIC 参数。消元后按顺序导出 624 个十进制状态整数，以逗号连接并计算 SHA-256，在 10 秒内提交即可得到：

```text
ACTF{482A17FD-17FA-872A-9DBB-41D715C0B266}
```

## 方法总结

- 核心技巧：把 MT19937 的状态更新与 temper 符号化为 $\mathrm{GF}(2)$ 线性映射，用大量 1 位截断输出恢复 19968 位状态。
- 识别信号：线性 PRNG 即使只泄露少量输出位，只要样本足够多且变换参数已知，仍可能形成满秩方程组。
- 复用要点：预计算矩阵与具体常量绑定；常量随机化后应保留符号建模框架，改用 M4RI 等位矩阵库现场建模和消元，并严格对齐状态更新时机。
