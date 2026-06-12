---
type: technique
tags: [ai-ml, technique]
skills: [ctf-ai-ml]
raw:
  - ../raw/ai-ml/llm-attacks.md
updated: 2026-05-21
---

# LLM Attacks

## 适用场景

模型行为、训练数据、推理接口、prompt、adversarial input 或 ML 参数是主要攻击面。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 题目涉及 classifier、LLM、embedding、fine-tuning、LoRA、membership 或 extraction。
- 成功条件取决于模型输出、置信度、梯度、token 或 prompt 行为。
- 需要查询策略、样本构造或模型分析。
- 题面或 raw 线索能落到这些关键词之一：Prompt Injection (Foundational)、LLM Attacks、Direct Prompt Injection、Indirect Prompt Injection、LLM Jailbreaking (Foundational)、Token Smuggling (Foundational)、Context Window Manipulation (Foundational)、Tool Use Exploitation (Foundational)。

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
| Prompt Injection (Foundational) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| LLM Attacks | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Direct Prompt Injection | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Indirect Prompt Injection | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| LLM Jailbreaking (Foundational) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Token Smuggling (Foundational) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Context Window Manipulation (Foundational) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Tool Use Exploitation (Foundational) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

## 常见陷阱

- 只按关键词跳页，没有先构造最小证据。
- 照搬 raw 中的一次性 payload，没有检查当前题的边界条件。
- 忽略相邻技巧之间的 pivot，导致在错误方向上继续投入时间。

## 关联技巧

- [adversarial-ml.md](adversarial-ml.md)
- [ml-model-inference-extraction-and-weight-analysis.md](ml-model-inference-extraction-and-weight-analysis.md)
- [ai-ml-tooling.md](ai-ml-tooling.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [RCTF2025-the-alchemists-cage-wp](../raw/misc/RCTF2025-the-alchemists-cage-wp.md) | 隐藏提示词泄露题的最小证据是模型能否复述历史对话、系统提示或被要求保护的 secret。 |
| [SU_easyLLMWP](../raw/ai-ml/SU_easyLLMWP.md) | LLM 对话约束题优先测试角色边界、输出格式约束和提示注入后的可验证回显。 |

## 原始资料

- [llm-attacks.md](../raw/ai-ml/llm-attacks.md)
