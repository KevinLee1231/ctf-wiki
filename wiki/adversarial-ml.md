---
type: family
tags: [ai-ml, family, adversarial-examples, evasion, poisoning, backdoor]
skills: [ctf-ai-ml]
raw:
  - ../raw/ai-ml/adversarial-ml.md
updated: 2026-06-12
---

# Adversarial ML

## 作用边界

本页是 adversarial examples、evasion、patch、poisoning 和 backdoor 检测 family。它负责从模型、输入约束、梯度可用性和评分反馈判断应走 FGSM/PGD/C&W、adversarial patch、black-box 查询、数据投毒还是后门触发器分析。

## 识别信号

- 成功条件是让分类器误判、通过模型认证、改变置信度、触发后门或规避检测。
- 题目给出模型文件、推理接口、训练数据、label、loss、梯度、输入范围或查询次数限制。
- 输入是图片、音频、特征向量或嵌入，且有像素/范数/格式/肉眼可读性约束。

## 最小证据

- 模型框架、输入 shape、归一化、label space、目标/非目标攻击要求。
- 是否 white-box：能否拿梯度、loss、权重；black-box 时反馈是 label、score 还是二值。
- 扰动约束：Linf/L2/L1、像素范围、mask、patch 位置、查询预算和提交格式。
- 成功样本能被本地或远端稳定复验。

## 路由表

| 证据 | 先验证 | 下一跳 |
|---|---|---|
| white-box 分类器 | loss、梯度和预处理是否一致 | FGSM/PGD/C&W |
| 只给 label/score API | 查询预算、score 粒度和随机性 | black-box search / NES / boundary |
| patch 可见区域 | patch 位置、大小和变换增强 | adversarial patch |
| MNIST/图像认证 | 输入 clamp、reshape 和归一化 | foolbox/hand-rolled gradient |
| poisoning/backdoor | 训练数据可控或触发器存在 | 触发器搜索/clean-label 判断 |
| 需要模型参数/权重 | 攻击面不是扰动而是模型恢复 | [ml-model-inference-extraction-and-weight-analysis.md](ml-model-inference-extraction-and-weight-analysis.md) |

## 合并与拆分结论

- 保留为 family：adversarial ML 覆盖多种攻击目标和反馈模型，适合二级分流。
- 不合并进 LLM 页：LLM prompt/tool 攻击的证据和验证方式不同。
- 暂不拆 FGSM/PGD/C&W 小页：当前 raw 更适合集中描述选择条件。

## 常见误判

- 本地预处理和远端不一致，导致样本本地成功远端失败。
- 梯度攻击后未 clamp 或量化，提交格式破坏扰动。
- Black-box 题没有估查询预算，过早使用高成本搜索。
- 只追求误分类，没有满足目标 label 或业务阈值。

## 关联页面

- [llm-attacks.md](llm-attacks.md)
- [ml-model-inference-extraction-and-weight-analysis.md](ml-model-inference-extraction-and-weight-analysis.md)
- [ai-ml-tooling.md](ai-ml-tooling.md)

## 原始资料

- [adversarial-ml.md](../raw/ai-ml/adversarial-ml.md)
