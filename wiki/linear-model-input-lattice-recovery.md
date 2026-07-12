---
type: technique
tags: [ai-ml, technique, linear-model, lattice, lwe, input-recovery, pytorch]
skills: [ctf-ai-ml, ctf-crypto]
raw:
  - ../raw/ai-ml/SUCTF2026-babyAIWP.md
updated: 2026-07-06
---

# Linear Model Input Lattice Recovery

## 适用场景

模型权重已知或可从 `state_dict` 中提取，网络没有激活函数或可展开成线性映射；公开输出是模 `q` 的线性结果再叠加小噪声。目标不是恢复模型参数，而是恢复输入字节、flag 向量或隐藏样本。

## 识别信号

- 网络形如 `Conv1d/Conv2d -> Flatten -> Linear`，没有 ReLU、pooling、softmax 等非线性。
- `model.pth`、checkpoint 或 zip 内 `data/*` 直接包含权重。
- 输出满足 `Y = A * X + E mod q`，其中 `E` 有小范围界。
- flag 格式、ASCII 范围、已知前后缀能显著收缩未知输入空间。

## 最小证据

- 已提取权重并重建总系数矩阵 `A`。
- 已确认模数、噪声界、输入长度、已知前后缀和字符范围。
- 候选解能回代，所有残差落在噪声界内。

## 解法骨架

1. 从 PyTorch checkpoint 读取卷积和线性权重；必要时直接按 zip + float32 结构解析。
2. 展开网络，把卷积窗口和全连接层合并成模 `q` 的矩阵 `A`。
3. 扣除已知 flag 字符贡献，只保留未知位置。
4. 将未知字符平移到中心区间，例如 `x_i = midpoint + u_i`，让 `u_i` 足够短。
5. 构造 CVP/BDD 格：前部承载模 `q` 关系，后部承载小变量约束。
6. LLL/BKZ 约化后用 Babai nearest plane 或 CVP 求近点。
7. 还原字符并回代验证噪声界。

## 关键变体

| 变体 | 判断方式 | 处理 |
|---|---|---|
| 欠定线性系统 + 小噪声 | 方程少于未知数，但字符范围和噪声很小 | 平移字符范围后做 CVP/Babai。 |
| 权重藏在 checkpoint | `model.pth` 是 zip/state_dict，不是缺失文件 | 直接提取 `data/*` 权重并按结构展开。 |
| 输出模数很大 | `q` 远大于 ASCII 和噪声 | 使用 centered mod 处理目标向量和残差。 |

## 常见陷阱

- 把这类题当 adversarial example，去优化扰动而不是展开线性关系。
- 忽略 checkpoint 内权重，误以为题目缺少模型参数。
- 没扣除已知 flag 前后缀，导致格维度和目标偏移过大。
- 解出字符后不回代检查噪声界。

## 关联技巧

- [ml-model-inference-extraction-and-weight-analysis.md](ml-model-inference-extraction-and-weight-analysis.md)
- [linear-model-parameter-recovery.md](linear-model-parameter-recovery.md)
- [lattice-and-lwe.md](lattice-and-lwe.md)
- [ai-ml-tooling.md](ai-ml-tooling.md)

## 原始资料

- [SUCTF2026-babyAIWP](../raw/ai-ml/SUCTF2026-babyAIWP.md)
