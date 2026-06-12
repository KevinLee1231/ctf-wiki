---
type: technique
tags: [ai-ml, technique]
skills: [ctf-ai-ml]
raw:
  - ../raw/ai-ml/adversarial-ml.md
updated: 2026-05-21
---

# Adversarial ML

## 适用场景

模型行为、训练数据、推理接口、prompt、adversarial input 或 ML 参数是主要攻击面。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 题目涉及 classifier、LLM、embedding、fine-tuning、LoRA、membership 或 extraction。
- 成功条件取决于模型输出、置信度、梯度、token 或 prompt 行为。
- 需要查询策略、样本构造或模型分析。
- 题面或 raw 线索能落到这些关键词之一：Adversarial Example Generation (FGSM, PGD, C&W)、FGSM (Fast Gradient Sign Method)、PGD (Projected Gradient Descent)、C&W (Carlini & Wagner) Attack、Adversarial Patch Generation、Evasion Attacks on ML Classifiers (Foundational)、Data Poisoning (Foundational)、Backdoor Detection in Neural Networks (Foundational)。

## 最小证据

- 已完成主方向判断，并确认本页技巧比相邻技巧更能解释当前证据。
- 至少有一个可复现输入、输出、文件结构、数学关系、协议行为或运行时状态。
- 能指出 raw 案例中哪一个变体与当前题最接近，以及不同点在哪里。

## 解法骨架

1. 固定输入格式和评分反馈。
2. 建立最小查询/训练/推理循环。
3. 选择 adversarial、extraction、prompt injection 等路径。
4. 用稳定样本验证结果。

## 关键变体

| 变体 | 复用重点 |
|---|---|
| Adversarial Example Generation (FGSM, PGD, C&W) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| FGSM (Fast Gradient Sign Method) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| PGD (Projected Gradient Descent) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| C&W (Carlini & Wagner) Attack | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Adversarial Patch Generation | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Evasion Attacks on ML Classifiers (Foundational) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Data Poisoning (Foundational) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Backdoor Detection in Neural Networks (Foundational) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| foolbox L1BasicIterativeAttack on Keras MNIST-Auth (nullcon 2019) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Hand-Rolled Keras FGSM via K.gradients (UTCTF 2019) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Adversarial Examples | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Gradient-Based Techniques | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

## 常见陷阱

- 只按关键词跳页，没有先构造最小证据。
- 照搬 raw 中的一次性 payload，没有检查当前题的边界条件。
- 忽略相邻技巧之间的 pivot，导致在错误方向上继续投入时间。

## 关联技巧

- [llm-attacks.md](llm-attacks.md)
- [ml-model-inference-extraction-and-weight-analysis.md](ml-model-inference-extraction-and-weight-analysis.md)
- [ai-ml-tooling.md](ai-ml-tooling.md)

## 原始资料

- [adversarial-ml.md](../raw/ai-ml/adversarial-ml.md)
