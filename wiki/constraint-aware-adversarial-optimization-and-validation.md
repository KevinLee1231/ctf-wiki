---
type: technique
tags: [ai-ml, adversarial-examples, bpda, eot, validation, technique]
skills: [ctf-ai-ml]
raw:
  - ../raw/ai-ml/ACTF2025-yolov-cls-wp.md
  - ../raw/ai-ml/UMDCTF2024-attack-of-the-worm-wp.md
  - ../raw/ai-ml/UMDCTF2024-the-worm-strikes-back-wp.md
  - ../raw/ai-ml/UMDCTF2026-flow-wp.md
  - ../raw/ai-ml/UMDCTF2026-dualflow-wp.md
  - ../raw/ai-ml/UMDCTF2026-rush-hour-wp.md
updated: 2026-07-27
---

# Constraint-Aware Adversarial Optimization and Validation

## 适用场景

白盒或可近似求导的模型允许优化输入，但成功条件还包含像素稀疏度、范数、随机变换、净化器、多个模型约束、序列动力学或提交文件格式。决定性问题不是能否得到梯度，而是梯度路径、约束投影和最终验证视图能否与服务端一致。

## 识别信号

- 本地张量已达到目标类别，落盘图片或远端判定却失败。
- 前向链含量化、阈值、随机裁剪、净化器或其它不可微步骤。
- 同一扰动必须跨图片、位置、旋转、模型或时间步稳定生效。
- 题目限制的是离散像素个数、真实文件字节、轨迹结果或似然阈值，而不只是连续张量上的 loss。

## 最小证据

- 完整复现输入解码、Resize/Crop、归一化、模型版本、随机性和输出判定。
- 明确可行域及其计算口径，例如 `L_inf`、修改像素数、patch 尺寸或多模型阈值。
- 保存优化态、量化落盘态和服务端同款重载态三组结果。
- 若使用近似梯度，分别验证真实前向结果和替代反向梯度没有被混为一谈。

## 解法骨架

1. 把服务端判定写成可重复执行的本地验证器，先用原始样本对齐预处理和输出。
2. 将目标 loss 与所有硬约束分开表达；每步更新后投影回真实可行域。
3. 不可微前向使用 BPDA，随机变换使用 EOT，多约束时将主梯度投影到活跃约束的切空间。
4. 序列环境按整段轨迹反向传播，并定期回到真实动力学复验。
5. 每轮只保存量化、编码后仍成功的候选；最终从磁盘重新读取并走完整验证链。

## 关键变体

| 变体 | 关键判断 |
|---|---|
| 稀疏像素攻击 | 约束按像素还是按通道分量计数；连续稀疏代理最终必须回到离散修改数。 |
| 通用 patch | 对样本与几何变换做期望优化，并用未参与训练的样本估计泛化率。 |
| BPDA | 前向严格保留真实防御，反向才替换为可用近似导数。 |
| 多模型约束 | 主梯度与约束梯度冲突时做正交投影，避免一步优化破坏另一模型阈值。 |
| 闭环序列攻击 | 小扰动沿动力学累积，最终成功必须由真实环境而非可微替身确认。 |

## 常见陷阱

- 只看优化张量的 logits，没有验证 PNG/JPEG 量化和重载后的结果。
- 把 BPDA 写成修改真实前向逻辑，得到服务端不会接受的样本。
- 用单张图片训练通用 patch，未覆盖位置、旋转和隐藏样本分布。
- 约束投影次序错误，使更新在多个约束之间来回震荡。
- 可微仿真已成功就停止，没有检查真实服务动力学的偏差。

## 关联技巧

- [adversarial-ml.md](adversarial-ml.md)
- [ml-model-inference-extraction-and-weight-analysis.md](ml-model-inference-extraction-and-weight-analysis.md)
- [ai-ml-tooling.md](ai-ml-tooling.md)

## 原始资料

- [ACTF2025-yolov-cls-wp.md](../raw/ai-ml/ACTF2025-yolov-cls-wp.md)
- [UMDCTF2024-attack-of-the-worm-wp.md](../raw/ai-ml/UMDCTF2024-attack-of-the-worm-wp.md)
- [UMDCTF2024-the-worm-strikes-back-wp.md](../raw/ai-ml/UMDCTF2024-the-worm-strikes-back-wp.md)
- [UMDCTF2026-flow-wp.md](../raw/ai-ml/UMDCTF2026-flow-wp.md)
- [UMDCTF2026-dualflow-wp.md](../raw/ai-ml/UMDCTF2026-dualflow-wp.md)
- [UMDCTF2026-rush-hour-wp.md](../raw/ai-ml/UMDCTF2026-rush-hour-wp.md)
