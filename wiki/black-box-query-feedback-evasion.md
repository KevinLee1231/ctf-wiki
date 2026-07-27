---
type: technique
tags: [ai-ml, adversarial-ml, black-box, evasion, query]
skills: [ctf-ai-ml]
raw:
  - ../raw/ai-ml/adversarial-ml.md
updated: 2026-07-27
---

# Black-Box Query-Feedback Evasion

## 适用场景

目标模型不可见或不可求梯度，但接口会返回 label、score、排名或稳定的二值通过结果；需要在查询预算内找到满足语义和格式约束的逃逸样本。

## 识别信号

- 只能远程提交样本，无法取得模型权重和真实预处理。
- 每次查询至少暴露可比较反馈，且相同输入的结果大体稳定。
- 可修改特征集合、距离预算、文件合法性或业务语义有明确边界。

## 最小证据

- 固定一个基线样本，重复查询以估计噪声和限流。
- 记录反馈粒度、查询成本、可修改维度及远端序列化方式。
- 至少保留一组“单一变更 -> 反馈变化”的可重放样本。

## 解法骨架

1. 复现远端输入编码，先排除缩放、量化和字段顺序差异。
2. 用单维扰动或分组消融估计敏感特征；只有 label 时采用边界搜索，有 score 时优先做坐标/NES 类优化。
3. 每轮投影回合法输入域，并把查询、候选和反馈落盘以便断点续跑。
4. 对最终样本重新序列化并多次复验，确认成功不是随机波动。

## 关键变体

- Label-only：先找跨界样本，再沿输入连线逼近决策边界。
- Score-based：按反馈差分估计方向，但必须计入噪声与查询预算。
- 结构化样本：变异器必须保持 PE、文本、音频或协议字段可被目标解析。

## 常见陷阱

- 把本地替代模型成功当作远端成功。
- 未测随机性，单次偶然命中被误认为稳定逃逸。
- 优化过程中破坏文件语义，模型虽然变化但提交已不可用。

## 关联技巧

- [adversarial-ml.md](adversarial-ml.md)
- [constraint-aware-adversarial-optimization-and-validation.md](constraint-aware-adversarial-optimization-and-validation.md)
- [model-backdoor-trigger-and-poisoning-analysis.md](model-backdoor-trigger-and-poisoning-analysis.md)

## 原始资料

- [adversarial-ml.md](../raw/ai-ml/adversarial-ml.md)
