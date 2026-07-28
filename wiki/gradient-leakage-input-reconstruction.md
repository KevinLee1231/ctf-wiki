---
type: technique
tags: [ai-ml, gradient-leakage, model-inversion, milp, technique]
skills: [ctf-ai-ml]
raw:
  - ../raw/ai-ml/UMDCTF2026-french-baguette-wp.md
updated: 2026-07-28
---

# Gradient Leakage Input Reconstruction

## 适用场景

服务或附件泄露单个样本产生的参数梯度，目标是恢复原始输入、标签或中间激活。该模式区别于“用梯度优化对抗样本”：这里的梯度本身是观测证据，网络结构决定可逆关系。

## 识别信号

- 给出一轮训练更新、每层权重梯度和偏置梯度，或可计算两次 checkpoint 差分。
- batch 很小，尤其是单样本，且网络中存在全连接层、池化层或离散输入约束。
- 输入来自二值图、有限字符集、像素格等强离散域。
- 目标不是训练一个近似模型，而是解释某次梯度对应的具体样本。

## 最小证据

- 层顺序、参数形状、激活函数、池化窗口和 batch 聚合方式已确定。
- 能确认梯度来自同一前向/反向过程，未混入未知动量、裁剪或多样本累积。
- 至少在一层验证恢复出的激活重新计算后能复现对应梯度。
- 离散输入的取值范围、尺寸和编码方式已固定。

## 解法骨架

1. 从最后一个可解析层开始，利用权重梯度和偏置梯度的比例恢复前一层激活。
2. 逐层逆推线性层和单调激活；遇到池化时把最大值位置或窗口选择写成约束。
3. 将卷积、池化和已恢复激活联立，先求连续候选。
4. 输入为二值或有限集合时，用 MILP/SMT 添加离散约束，消除连续解的多义性。
5. 用原模型对恢复输入重做前向和反向，比较全部已知梯度而不是只看视觉相似。

## 关键变体

- 多样本 batch 会把外积求和，直接比例关系失效，需要额外先验或优化式反演。
- 含 BatchNorm、dropout、梯度裁剪或优化器状态时，必须先还原训练态语义。
- 池化不是普通可逆算子，但局部最大值位置可由梯度和离散约束共同确定。
- checkpoint 差分只有在学习率和优化器更新规则已知时才能还原真实梯度。

## 常见陷阱

- 把梯度泄漏误当作普通模型参数反演，直接对像素做无结构梯度下降。
- 忽略 bias gradient，丢失恢复线性层输入的直接比例。
- 只匹配一层梯度，得到无法解释其它层的伪解。
- 连续优化结果接近原图就停止，没有利用二值或字符集约束收紧为唯一解。

## 关联技巧

- [ml-model-inference-extraction-and-weight-analysis.md](ml-model-inference-extraction-and-weight-analysis.md)
- [linear-model-discrete-input-recovery.md](linear-model-discrete-input-recovery.md)
- [linear-model-parameter-recovery.md](linear-model-parameter-recovery.md)
- [ai-ml-tooling.md](ai-ml-tooling.md)

## 原始资料

- [UMDCTF2026-french-baguette-wp.md](../raw/ai-ml/UMDCTF2026-french-baguette-wp.md)
