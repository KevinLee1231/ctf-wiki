# lights-out

## 题目简述

题目提供一个 Minecraft 世界，其中 256 个 lever、256 个灯泡和巨大的 observer 红石布线实现了 Lights Out。每个 lever 的按下状态是一个二进制变量，每个灯泡的初始亮灭是一个方程右端；红石路径定义某个 lever 会翻转哪些灯。源码表明 flag 恰为 32 字节，即 256 bit，且构造时特意保证连接矩阵满秩。因此主要障碍是从世界文件恢复该线性变换并在 $\mathrm{GF}(2)$ 上求解，分类为 reverse，而非 Minecraft 操作题。

## 解题过程

### 从红石世界抽取矩阵

世界生成器对每个连接保存 `(btn, eqn)`，含义是第 `btn` 个 lever 影响第 `eqn` 个灯泡；每个方程有 16 条随机连接和一条对角连接。生成器构建的矩阵满足：

$$
M_{\text{eqn},\text{btn}}=1\quad\Longleftrightarrow\quad(\text{btn},\text{eqn})\text{ 存在一条 observer 路径}.
$$

不必在客户端手动追数千条线。解析 `region/r.0.0.mca` 的方块状态后，读取 observer 的 `facing` 属性：每条输入路径从按钮所在的 $z=\text{btn}$ 通道进入 256×256×256 布线立方体，并在输出端到达 $z=\text{eqn}$ 的灯泡回路。遍历所有起点、记录终点即可填充 `256×256` 的 $M$。初始灯泡位于 UI 平台，对应向量 $b$；亮为 1，灭为 0。

源码中的 `world.py` 还给出了物理布局的相对关系：每行依次放置 lever、初始 bulb、lamp、输入竖线、布线立方体和返回线。因此即使没有生成期的 `connections` 数组，也能以方块的 `facing` 图恢复同一矩阵。

### 在 GF(2) 中求按键向量

按下向量 $v$ 应使初始状态清零，故解线性方程：

$$
Mv=b\pmod2.
$$

构造器已验证 `rank(M)=256`，所以没有多解或无解的歧义。用任意 GF(2) 高斯消元即可；例如在已取得 $M$ 与 $b$ 后：

```python
import numpy as np
import galois

GF2 = galois.GF(2)
presses = np.linalg.solve(GF2(M), GF2(b))
bits = "".join(map(str, map(int, presses.tolist())))
flag = int(bits, 2).to_bytes(32, "big").decode()
print(flag)
```

这里 bit 顺序必须按构造器使用的“每个 flag byte 的高位到低位”顺序还原，最后以大端整数转回 32 字节。解出的 lever 向量不仅能使游戏中的 256 个灯全灭，也恢复：

```text
grey{addin_redstone_2_my_rEsumE}
```

## 方法总结

- 核心技巧：把物理红石路径抽象为二元矩阵，Lights Out 立即变成 $\mathrm{GF}(2)$ 的线性方程组。
- 识别信号：题面包含大量独立开关/灯，源码或结构显示 XOR/翻转传播，并刻意保证矩阵 rank 时，应优先线性化而非手工试按。
- 复用要点：从容器或世界文件重建图时，先确定边的方向和输入/输出坐标；求解后同时验证“全灯熄灭”和 bit-to-byte 的字节序，避免仅得到看似合理的乱码。
