---
type: technique
tags: [ai-ml, backdoor, poisoning, trigger, model-analysis]
skills: [ctf-ai-ml]
raw:
  - ../raw/ai-ml/adversarial-ml.md
updated: 2026-07-27
---

# Model Backdoor Trigger and Poisoning Analysis

## 适用场景

模型在干净输入上表现正常，却可能因局部 trigger、特定 token、训练样本污染或异常神经元激活而稳定偏向攻击者目标。

## 识别信号

- 某个小区域、短 token 序列或固定特征组合会强制改变预测。
- 训练集可控、来源混杂，或题目直接给出 clean/poisoned 数据与模型。
- 异常类别的 activation、梯度或置信度分布形成独立簇。

## 最小证据

- 同一批 clean 样本在有无候选 trigger 时的预测对照。
- Trigger 的位置、尺寸、强度和目标标签是否可迁移到多个样本。
- 至少一种模型内部或数据分布证据，而不只是一张偶然误分类图片。

## 解法骨架

1. 建立 clean accuracy 与按类别 confusion matrix 基线。
2. 对局部 patch、token 或特征子集做遮挡、反演或梯度引导搜索。
3. 比较 clean/trigger 输入的中间 activation，定位异常簇或高影响神经元。
4. 在未参与搜索的样本上验证 attack success rate，并区分后门与普通对抗脆弱性。

## 关键变体

- Clean-label poisoning：标签看似正常，需从特征聚类和训练影响识别。
- Patch/backdoor：固定局部模式跨样本稳定触发。
- NLP/LLM trigger：触发器可能是 token 序列、模板或上下文位置。

## 常见陷阱

- 只验证单一样本，无法证明触发器具有稳定性。
- 将类别本身的显著特征误判成后门神经元。
- 忽略训练与推理预处理差异，导致 trigger 复验失败。

## 关联技巧

- [adversarial-ml.md](adversarial-ml.md)
- [black-box-query-feedback-evasion.md](black-box-query-feedback-evasion.md)
- [constraint-aware-adversarial-optimization-and-validation.md](constraint-aware-adversarial-optimization-and-validation.md)

## 原始资料

- [adversarial-ml.md](../raw/ai-ml/adversarial-ml.md)
