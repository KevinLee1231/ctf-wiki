# bi0sCTF 2024 - Katanaverse 0.0

## 题目简述

题目给出一个本地驱动程序和自定义 VM 的代码转储。输入先被规范化为 `bi0sCTF{...}` 并做 Base64 编码，再由 VM 拆成比特矩阵。每组比特选择 16 种量子门组合之一，在 Bloch 球上生成 12 个连续状态；这些状态之间的距离构成加权图，随后通过 QAOA/最大割结果生成字节，最后再经过 AES 与内置目标比较。

二进制并不是简单的逐字节 flag checker。可行做法是先还原 VM 指令与比特流，再在本地用经典矩阵模拟单量子比特门，并对只有 12 个顶点的最大割问题直接枚举；不必真正依赖量子硬件。

## 解题过程

### 还原 VM 与输入编码

`dump.dmp` 以四个十进制字符表示一条 opcode。主程序每次读取 4 字节并转成整数，分派到 `1200` 至 `1243` 的指令，包括寄存器移动、算术、异或、移位、栈操作、条件跳转与输入。先按源码中的 switch 表写出反汇编器，顺着 PC、ZF、SP、MP 等寄存器模拟，就能确定：

- 输入字符串被包装为标准 flag 格式；
- VM 对 Base64 字符逐位处理；
- 每个字符最终形成 6 行 8 位 `classified` 数据；
- 每行的高、低四位分别编码一个 $0$ 到 $15$ 的门编号。

因此每个字符产生 12 个操作编号：

```python
operations = []
for row in classified_group:
    high = 8 * row[0] + 4 * row[1] + 2 * row[2] + row[3]
    low = row[4] + 2 * row[5] + 4 * row[6] + 8 * row[7]
    operations.extend([high, low])
```

程序还从最初的若干行构造 PRNG seed，并要求触发字符为 `r`。按相同 seed 重放 `rand()` 后，利用内置 `conditions` 检查每组六行的特定位关系，可以在进入量子状态计算前大量剪枝。

### 经典模拟量子门

16 个操作最终都是绕 $x,y,z$ 轴的固定角旋转。没有必要连接量子后端：从初态 Bloch 向量

$$
v_0=(0,0,1)^T
$$

开始，按操作序列左乘对应的三维旋转矩阵即可。例如

$$
R_x(\theta)=
\begin{pmatrix}
1&0&0\\
0&\cos\theta&-\sin\theta\\
0&\sin\theta&\cos\theta
\end{pmatrix}.
$$

$R_y,R_z$ 同理。源码每次组合若干固定角度，并把小于 $10^{-5}$ 的分量归零、其余分量保留 5 位小数；solver 必须复现这一舍入，否则后续边权可能相差 1。

对每组 12 个状态 $v_i$，源码按固定 41 对顶点建立边，权重为

$$
w_{ij}=\operatorname{round}\left(10\lVert v_i-v_j\rVert_2\right).
$$

### 用经典穷举替代 QAOA

所谓 QAOA 输出实际是在近似求 12 点加权最大割。总状态数只有 $2^{12}=4096$，直接枚举每个二进制划分 $x$，计算

$$
C(x)=\sum_{i,j}w_{ij}x_i(1-x_j)
$$

即可稳定得到最优比特串，避免量子后端采样误差：

```python
best_bits, best_cost = None, -1
for mask in range(1 << 12):
    bits = [(mask >> i) & 1 for i in range(12)]
    cost = sum(w[i][j] * bits[i] * (1 - bits[j])
               for i in range(12) for j in range(12))
    if cost > best_cost:
        best_bits, best_cost = bits, cost
```

将每组的 12 位结果按源码位序拆成两个 6 位值并加 64，得到送入 AES 的字节。AES 密钥来自 `classified` 的前 16 行；复现加密并与内置 32 字节 `unk_0` 比较，就能验证候选。官方求解资料给出的最终内容为 `QuBitJugglr`，包装后为 `bi0sCTF{QuBitJugglr}`。

## 方法总结

本题的复杂度主要来自多层表示：VM 字节码、Base64 比特、量子门编号、Bloch 球坐标、最大割和 AES。逐层保持源码的位序、随机数序列与舍入规则，就能把问题化为确定性的经典计算。QAOA 只是包装；12 个顶点完全可以穷举，从而得到可复现且不受量子采样噪声影响的结果。
