# SUCTF2026-babyAI

## 题目简述

附件包含生成脚本 `task.py` 和 PyTorch 权重 `model.pth`。生成脚本把 41 字节 flag 输入一个没有 bias、没有激活函数的小网络：

```python
Conv1d(1, 1, kernel_size=3, stride=2, bias=False)
Linear(20, 15, bias=False)
```

卷积核和全连接权重是模数范围内的整数，最终输出加入独立小噪声再模 $q$：

```text
q = 1_000_000_007
n = 41
m = 15
|e_i| <= 160
```

公开结果是 15 维向量 $Y$。`model.pth` 并非无关的模型附件：它完整保存了两层权重。由于网络全程线性，展开卷积后可得到一个带小误差的模线性系统

$$
Y\equiv WX+E\pmod q,
$$

其中 $W$ 是已知 `15×41` 矩阵，$X$ 是 flag 的 ASCII 字节，$E$ 是小噪声。这是一个带有短秘密和短误差的 LWE/BDD 型格问题。

## 解题过程

### 从模型权重构造等效矩阵

一维卷积产生 20 个输出：

$$
c_i=w_0x_{2i}+w_1x_{2i+1}+w_2x_{2i+2},\qquad 0\le i<20.
$$

全连接层再计算

$$
Y_j=\sum_{i=0}^{19}F_{j,i}c_i+e_j\pmod q.
$$

将卷积写成变体 Toeplitz 矩阵 $M_{conv}\in\mathbb{Z}^{20\times41}$，即可合并两层：

$$
W=F M_{conv}\bmod q.
$$

用 PyTorch 读取权重时，应以附件中实际保存的 `float32` 整数值为准：

```python
state = torch.load("model.pth", map_location="cpu", weights_only=True)
w_conv = [int(v) for v in state["conv.weight"].squeeze().tolist()]
w_fc = [[int(v) for v in row] for row in state["fc.weight"].tolist()]

M_conv = np.zeros((20, 41), dtype=object)
for i in range(20):
    M_conv[i, 2 * i:2 * i + 3] = w_conv
W_total = (np.array(w_fc, dtype=object) @ M_conv) % q
```

没有 PyTorch 时，`.pth` 仍是 ZIP 容器。本题中 `model/data/0` 是 3 个小端 `float32` 卷积权重，`model/data/1` 是 300 个全连接权重；可用 `zipfile` 和 `struct.unpack` 读取后转成整数。

### 为什么欠定系统仍可恢复

只有 15 个方程却有 41 个未知字节，普通高斯消元不可能唯一确定 $X$。但本题还有强约束：

- flag 字节是很小的可打印 ASCII 值；
- flag 格式以 `SUCTF{` 开头、以 `}` 结尾；
- 每个误差绝对值至多 160；
- 模数约为 $10^9$，远大于秘密和误差。

因此正确解对应格中的异常短向量。直接把 ASCII 值放入格时，41 个约 `50–120` 的分量会使目标向量偏长；官方解法先做中心化。

### 中心化秘密

取可打印字符中心值 $\mu=80$，令

$$
X=\Delta X+\mu\mathbf{1}.
$$

则

$$
Y'=Y-\mu W\mathbf{1}\equiv W\Delta X+E\pmod q.
$$

原本约为几十到一百多的 ASCII 值被换成围绕 0 的偏移量，目标短向量从 $(X,-E,C)$ 变成 $(\Delta X,-E,C)$，显著降低范数。

### 官方路线：57 维嵌入格与 BKZ

取 $C=160$，构造维数 $n+m+1=57$ 的行格基：

$$
B=
\begin{pmatrix}
I_n & W^T & 0\\
0 & qI_m & 0\\
0 & -Y'^T & C
\end{pmatrix}.
$$

存在整数向量 $(\Delta X,K,1)$，使它与 $B$ 的线性组合为

$$
(\Delta X,-E,C),
$$

该向量的秘密偏移、误差和嵌入常数都很小。用 BKZ 约化后，枚举末坐标为 $\pm C$ 的短行，统一符号，再把前 41 项加回 $\mu$：

```python
mu = 80
Y_prime = vector(ZZ, (Y_vec - mu * W * vector(ZZ, [1] * n)) % q)

dim = n + m + 1
B = Matrix(ZZ, dim, dim)
for i in range(n):
    B[i, i] = 1
    for j in range(m):
        B[i, n + j] = W[j, i]
for i in range(m):
    B[n + i, n + i] = q
    B[dim - 1, n + i] = -Y_prime[i]
B[dim - 1, dim - 1] = 160

for block_size in [15, 20, 25, 30]:
    reduced = B.BKZ(block_size=block_size)
    for row in reduced:
        if abs(row[-1]) != 160:
            continue
        vec = row if row[-1] == 160 else -row
        candidate = bytes(int(v + mu) for v in vec[:n])
        if candidate.startswith(b"SUCTF{"):
            print(candidate, list(vec[n:n + m]))
```

逐级增加 block size，可以先用较低成本尝试；噪声扩大到 160 后，单纯 LLL 不一定稳定，BKZ 是官方更稳妥的方案。

### 参赛队变体：已知格式消元后做 LLL + Babai

总 PDF 给出的解法先把 `SUCTF{` 和末尾 `}` 这 7 个已知字节对 $Y$ 的贡献减掉，只为剩余 34 个未知字符建格。再取中心值 79，使未知偏移大致位于 `[-47,47]`，构造：

- 15 列 $q e_i$，处理模数倍数；
- 每个未知字符一列 $(W'_{:,j},e_j)$；
- 目标为 $(Y'-W'\cdot79,0,\ldots,0)$。

对该基做 LLL 约化，再用 Babai nearest plane 找目标最近格点，就能恢复字符偏移。这条路线利用了更多格式先验，维度更低；本题实例上能够直接成功，但一般不如更深的 BKZ 稳定。

### 回代验证

无论使用哪种格路线，都不能只凭可打印字符串或前缀判断成功。应按原系统逐行计算中心化残差：

```python
errors = []
for row in range(15):
    value = sum(int(W_total[row, i]) * flag[i] for i in range(41)) % q
    diff = (value - Y[row]) % q
    if diff > q // 2:
        diff -= q
    errors.append(diff)
assert max(map(abs, errors)) <= 160
```

恢复结果为：

```text
SUCTF{PyT0rch_m0del_c4n_h1d3_LWE_pr0bl3m}
```

对应 15 个残差为：

```text
[-53, 105, 105, -55, 9, -17, 65, -2, 140, -111, 101, 76, 81, 126, -109]
```

每一项均满足生成脚本的噪声界。

## 方法总结

- 核心技巧：把无激活的卷积/线性网络展开为已知模线性矩阵，再用中心化嵌入格恢复短 ASCII 秘密。
- 识别信号：模型权重公开、网络只有线性算子、输出在大模数下加入远小于模数的噪声，且秘密字符范围很小。
- 复用要点：先从真实 `.pth` 提取权重并核对矩阵维度；中心化能显著缩短目标向量。LLL/Babai 可结合已知格式降维，噪声更大时应转向 BKZ；最终必须回代验证每个残差都满足噪声界。
