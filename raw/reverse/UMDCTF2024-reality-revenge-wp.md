# reality's revenge

## 题目简述

本题沿用 `reality` 的浮点指令虚拟机，但修复了第一版的单边误差比较。程序接收两个 $15\times15$ 矩阵 $Q$、$R$，要求 $Q$ 正交、$R$ 为上三角矩阵，并满足 $QR=A$。全零矩阵不再可用，必须提交内置矩阵 $A$ 的真实 QR 分解。

## 解题过程

虚拟机仍把第 $i$ 条指令放在

$$
\frac{\sin(i)}{i+1}
$$

对应的浮点地址上，并用 `MOV`、`ADD`、`SUB`、`MUL`、`CAS` 和 `JMP` 拼出循环。先根据 `map.c` 中的操作码语义恢复 `codegen.py` 的标签程序，比直接追逐随机排列后的双向链表更清楚。

第一版会直接比较有符号残差。复仇版在正交性和乘积检查中增加了如下等价操作：

```text
sign = SGN(error)
error = error * sign
```

因此实际比较量变为 $|\text{error}|$。完整条件是：

$$
\left|(QQ^{T})_{ij}-\delta_{ij}\right|<10^{-5},
$$

$$
R_{ij}=0\quad(i>j),
$$

$$
\left|(QR)_{ij}-A_{ij}\right|<10^{-5}.
$$

仓库的 `solve.npz` 保存了与发布二进制匹配的 `a`、`q`、`r`。其中数值满足：

```text
max(abs(Q @ Q.T - I)) = 3.33e-16
max(abs(Q @ R - A))   = 1.78e-15
max(abs(tril(R, -1))) = 0
```

官方 `solve.py` 只是把同一组 225 个 $Q$ 元素和 225 个 $R$ 元素展开成硬编码列表。可以保留计算来源，用更短的脚本按行发送：

```python
import numpy as np
from pwn import *

data = np.load("solve.npz")
q = data["q"]
r = data["r"]

assert q.shape == (15, 15)
assert r.shape == (15, 15)
assert np.max(np.abs(q @ q.T - np.eye(15))) < 1e-5
assert np.max(np.abs(np.tril(r, -1))) < 1e-5
assert np.max(np.abs(q @ r - data["a"])) < 1e-5

# 远程环境改为 remote(host, port)
io = process("./reality")
for value in np.concatenate((q.ravel(), r.ravel())):
    io.sendline(repr(float(value)).encode())

print(io.recvall().decode())
```

输出：

```text
Welcome to reality.
UMDCTF{guys please dont break reality again at flocto anyways hope you liked the qr decomposition}
```

如果只有发布二进制而没有 `solve.npz`，也可以在还原虚拟机后提取初始化阶段写入正地址区的 225 个 $A_{ij}$ 常量，再执行：

```python
q, r = np.linalg.qr(a)
```

发送顺序必须是 $Q$ 按行展开在前、$R$ 按行展开在后。程序内部以 `float` 保存输入，保留 Python 浮点数的完整十进制表示即可把舍入误差控制在阈值内。

## 方法总结

复仇版没有引入新的密码学或利用点，而是把原题漏掉的绝对值补全，使校验真正等价于 QR 分解。解题关键仍是先剥掉浮点地址虚拟机，确认输入布局和三个矩阵约束；随后提取 $A$ 并调用稳定的数值线性代数实现即可。提交前同时验证正交误差、下三角元素和重构误差，可以避免因矩阵顺序或展开顺序错误而无输出。
