# SUCTF2026-我不是神偷

## 题目简述

本题要求恢复一个无激活函数的 PyTorch 模型。线上真实结构为：

```text
Input(1×1×22×22)
 -> Conv2d(1, 1, 4×4)
 -> Conv2d(1, 1, 4×4)
 -> Flatten(256)
 -> Linear(256, 256)
```

服务提供 `/predict` 返回完整的 256 维输出，`/flag` 则加载选手提交的 `state_dict` 并逐参数比较。官方解法依赖附件中的强化前模型 `model_base.pth`：强化后仍复用了其中的 `linear.weight` 和 `linear.bias`，只需从黑盒输出恢复两层新卷积。

题目有意要求“先看行为，再看命名”。附件 `app.py` 写的是 `8×8`、`7×7` 两层卷积，briefing 也只给出历史 warmup 记录 `legacy conv.bias=-4, conv1.bias=8`；它们都不是线上完整快照。当前模型的两个已知偏置来自题目 README：

```text
conv.bias  = -5.640393257141113
conv1.bias = -4.398319721221924
```

归档部署源码与官方 WP 还有一处版本差异：源码实际使用 `0.0001` 作为逐元素误差阈值，官方文字和比赛期参赛队材料写的是 `0.01`。复现时应以更严格的 `1e-4` 为目标，并在提交前用本地前向误差验证候选模型。

## 解题过程

### 从行为确认结构

向 `/flag` 提交缺键或形状错误的模型，可以从 `load_state_dict` 报错确认真实参数为：

```text
linear.weight  (256, 256)
linear.bias    (256,)
conv.weight    (1, 1, 4, 4)
conv.bias      (1,)
conv1.weight   (1, 1, 4, 4)
conv1.bias     (1,)
```

再探测 `/predict` 可知正常单图输入是 `22×22`。两次无 padding、步长为 1 的 `4×4` 卷积使空间尺寸依次变为 `19×19`、`16×16`，恰好展平为 256 维。

源码中的 `x.view(-1)` 没有保留 batch 维，因此只要卷积后总元素数恰为 256，一些异常 batch/尺寸组合也可能进入线性层；但预期解法不依赖该行为，使用标准单图基向量查询更稳定。`torch.load(..., weights_only=True)` 也使常规 pickle 反序列化 RCE 不是本题主线。

### 恢复整体仿射映射

整个网络没有 ReLU、池化或 BN，展平输入 $x\in\mathbb{R}^{484}$ 与输出 $y\in\mathbb{R}^{256}$ 满足

$$
y=Ax+b.
$$

查询一次全零输入得到 $b=f(0)$，再逐个查询 484 个标准基向量：

$$
A_{:,i}=f(e_i)-f(0).
$$

核心实现如下；把每列响应落盘缓存可以防止网络中断后重跑全部 485 次请求。

```python
def extract_affine(query):
    zero = np.zeros((1, 1, 22, 22), dtype=np.float32)
    bias = query(zero)
    matrix = np.empty((256, 22 * 22), dtype=np.float64)
    for i in range(22 * 22):
        sample = zero.copy()
        sample.reshape(-1)[i] = 1.0
        matrix[:, i] = query(sample) - bias
    return matrix, bias
```

### 用基线模型剥离线性层

从 `model_base.pth` 读取共享线性层：

```python
base = torch.load("model_base.pth", map_location="cpu", weights_only=True)
W = base["linear.weight"].numpy().astype(np.float64)
D = base["linear.bias"].numpy().astype(np.float64)
```

设两层卷积合成后的线性映射为 $S$、其常数输出为 $c$，则

$$
A=WS,\qquad b=Wc+D.
$$

因此：

```python
S = np.linalg.solve(W, A)
c_vec = np.linalg.solve(W, b - D)
effective_bias = float(c_vec.mean())
```

`c_vec` 的 256 个分量理论上应相同。检查其标准差接近 0，既能验证线性层确实被复用，也能及时发现查询噪声、输入形状或基础模型不匹配。

### 提取等效 `7×7` 卷积核

两个 `4×4` 核串联后的感受野为 `4+4-1=7`。$S$ 的第 $k$ 行 reshape 为 `22×22` 后，只应在对应输出位置的 `7×7` 窗口内非零。枚举 256 行、按输出坐标切出这些窗口并取平均即可得到等效核 $K$：

```python
patches = []
for row_index in range(256):
    top, left = divmod(row_index, 16)
    row = S[row_index].reshape(22, 22)
    patches.append(row[top:top + 7, left:left + 7])
K = np.stack(patches).mean(axis=0)
```

应同时计算两个结构残差：各 patch 相对平均核的最大离散度，以及把 $K$ 平移回 256 行后与 $S$ 的相对误差。两者都很小时，才能确认提取出的确是平移不变卷积结构，而不是数值拟合偶然成功。

### 将 `7×7` 核分解为两个 `4×4` 核

设第一、第二层卷积核为 $K_1,K_2$，其等效核满足 full convolution：

$$
K=K_1*K_2.
$$

可把一个核的首元素固定为 1，去掉一个显式缩放自由度，然后用 `scipy.optimize.least_squares` 多次随机重启，最小化 49 个等效核元素的残差：

```python
def full_convolution_4x4(a, b):
    out = np.zeros((7, 7), dtype=np.float64)
    for i in range(4):
        for j in range(4):
            out[i:i + 4, j:j + 4] += a[i, j] * b
    return out

def objective(params):
    a = np.empty((4, 4), dtype=np.float64)
    a.flat[0] = 1.0
    a.flat[1:] = params[:15]
    b = params[15:].reshape(4, 4)
    return (full_convolution_4x4(a, b) - K).ravel()
```

即使 $K$ 已完全拟合，分解仍存在

$$
(sK_1)*(K_2/s)=K
$$

的缩放歧义，而且由于 full convolution 交换律，还存在两层顺序歧义。

### 用偏置约束消除缩放歧义

全零输入经过第一层后是常数 `conv.bias`。设第二层核为 $K_2$，两层卷积合成后的常数为

$$
c=\text{conv1.bias}+\text{conv.bias}\sum K_2.
$$

前面已从整体仿射偏置恢复出 `effective_bias`，所以目标第二层核元素和为

$$
\sum K_2=\frac{\text{effective_bias}-\text{conv1.bias}}{\text{conv.bias}}.
$$

对某次基础分解 `(first, second)`，按下式重缩放即可满足偏置约束：

```python
expected_sum = (effective_bias - CONV1_BIAS) / CONV_BIAS
scale = second.sum() / expected_sum
first = first * scale
second = second / scale
```

分别尝试 `(first, second)` 与 `(second, first)`。每个候选都先在若干随机输入上比较

```text
candidate(x)  与  A @ x + b
```

的最大绝对误差，再把通过本地验证的 `state_dict` 提交给 `/flag`。这比单纯把远端非 400 响应当成正确结果更可靠。

### 参赛队的历史服务替代路线

总 PDF 记录了比赛期间仍可访问上一题单卷积服务的情形。若没有 `model_base.pth`，可以先对历史服务做 362 次 basis query，恢复其整体映射 $M_2$；利用已恢复的 `4×4` 卷积核构造平移矩阵 $H_2$，再求

$$
W=M_2H_2^T(H_2H_2^T)^{-1},\qquad
D=b_2-W(4\mathbf{1}).
$$

随后回到当前服务继续执行上面的线性层剥离、`7×7` 核提取与分解。这条路线依赖比赛期历史服务仍在线，不如官方的本地 `model_base.pth` 路线稳定，但解释了参赛队总 PDF 中“先打通旧服务”的步骤。

参赛队实测恢复出的候选通过了远端校验，得到：

```text
SUCTF{v3r1fy_b3h4v10r_n0t_h1st0ry_7a4c9d21}
```

## 方法总结

- 核心技巧：用 basis query 恢复纯线性网络的整体仿射映射，再借共享线性层拆出等效卷积并做双线性因式分解。
- 识别信号：推理接口返回完整向量、网络无激活函数、附件提供基线模型或历史服务、提交接口逐参数比较权重。
- 复用要点：先用形状和残差验证每个假设；用已知 bias 消除卷积因式分解的缩放歧义，同时枚举层顺序。历史说明、附件命名和实际部署冲突时，应记录版本边界并以可验证行为为准。
