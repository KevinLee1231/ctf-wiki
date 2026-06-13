---
type: family
tags: [ai-ml, model-extraction, inference, weight-analysis, family]
skills: [ctf-ai-ml]
raw:
  - ../raw/ai-ml/model-attacks.md
updated: 2026-06-13
---

# ML Model Inference, Extraction and Weight Analysis

## 适用场景

本页是 AI/ML 模型攻击 family。模型本身是主要证据：需要通过推理接口、输出置信度、任务语义、模型权重、embedding、encoder、LoRA adapter 或训练/推理参数恢复 flag。若核心是 prompt injection，转到 [llm-attacks.md](llm-attacks.md)；若核心是扰动样本欺骗分类器，转到 [adversarial-ml.md](adversarial-ml.md)。

## 识别信号

- 附件包含模型文件、权重、配置、推理服务、输入输出样本、embedding、encoder 或 adapter。
- 成功条件取决于模型输出、隐藏类别、相似度、置信度、梯度、参数差异或训练数据痕迹。
- 服务允许反复查询，且反馈可以被统计、拟合、搜索或反演。
- 题目表面像业务逻辑，但关键判断由模型推理结果或模型内部状态决定。

## 最小证据

- 固定输入格式、预处理、输出字段和评分/判定阈值。
- 记录至少一组可复现输入输出，并说明反馈是离散标签、概率、embedding、loss、文本还是 side channel。
- 若有模型文件，先确认框架、层结构、权重维度、checkpoint/config 关系和是否存在 adapter/patch。
- 能说明当前路径是 query extraction、model inversion、weight diff、encoder collision、membership inference，还是普通 adversarial 样本。

## 解法骨架

1. 先做 I/O 契约：输入类型、shape、归一化、tokenizer、输出字段和限制次数。
2. 如果能查询服务，构造最小样本集，观察标签、概率、长度、错误和边界反馈。
3. 如果有权重或 checkpoint，检查 config、state_dict、异常层、adapter 合并和权重差分。
4. 根据反馈选择路线：抽取近似模型、反演输入、搜索碰撞、恢复隐藏类别、推断训练样本或合并/抵消权重扰动。
5. 输出可复验样本或脚本，证明模型产生目标分类、目标文本、目标 embedding 或恢复出的 flag。

## 关键变体

| 变体 | 复用重点 |
|---|---|
| Query-based model extraction | 查询预算、输入覆盖、输出置信度和近似模型评估是核心；先建立可重复采样策略。 |
| Model inversion | 输出概率、embedding 或 loss 可提供梯度/优化目标；先复现预处理和约束范围。 |
| Weight analysis / perturbation | 对比 checkpoint、adapter 或权重补丁，找异常矩阵、偏置、符号翻转和可逆扰动。 |
| Encoder collision | 如果成功条件是相同 embedding/latent，优先分析归一化、哈希/量化和距离阈值。 |
| Membership inference | 需要区分训练样本和非训练样本反馈差异，避免把随机高置信误判为成员证据。 |
| Task-semantic inference | 部分题没有复杂 ML 攻击，关键是把输入输出约束形式化成搜索、分类或规则恢复问题。 |

## 常见陷阱

- 没有复现预处理就直接优化输入，导致本地结果和服务不一致。
- 把 LLM prompt 绕过放到模型抽取页，造成路线混乱。
- 忽略查询预算和反馈粒度，选择了不可能收敛的攻击路线。
- 只看模型结构，不检查 checkpoint/config/tokenizer 是否匹配。
- 对中文题名或业务叙事过拟合，没有把任务抽象成可测试的输入输出约束。

## 关联技巧

- [adversarial-ml.md](adversarial-ml.md)
- [llm-attacks.md](llm-attacks.md)
- [ai-ml-tooling.md](ai-ml-tooling.md)

## WP 案例沉淀

本节只抽取 raw WP 中可复用的识别信号和下一跳，不替代原始题解正文。

| Raw WP | 复用信号 | 下一跳 |
|---|---|---|
| [LilacCTF2026-residue-wp](../raw/ai-ml/LilacCTF2026-residue-wp.md) | 模型输入或特征边界可控，优先构造最小扰动并验证分类/评分差异。 | [adversarial-ml.md](adversarial-ml.md) |
| [SU_谁是小偷WP](../raw/ai-ml/SU_谁是小偷WP.md) | 推理行为或任务语义是主线，先把输入输出抽象成可测试约束。 | [ml-model-inference-extraction-and-weight-analysis.md](ml-model-inference-extraction-and-weight-analysis.md) |
| [SU_我不是神偷WP](../raw/ai-ml/SU_我不是神偷WP.md) | 推理行为或任务语义是主线，先把输入输出抽象成可测试约束。 | [ml-model-inference-extraction-and-weight-analysis.md](ml-model-inference-extraction-and-weight-analysis.md) |
| [SU_babyAIWP](../raw/ai-ml/SU_babyAIWP.md) | 模型输入或特征边界可控，优先构造最小扰动并验证分类/评分差异。 | [adversarial-ml.md](adversarial-ml.md) |
| [SU_easyLLMWP](../raw/ai-ml/SU_easyLLMWP.md) | LLM 对话约束和 prompt 边界，优先找提示注入、角色泄露和输出格式绕过。 | [llm-attacks.md](llm-attacks.md) |
| [SU_theifWP](../raw/ai-ml/SU_theifWP.md) | 推理行为或任务语义是主线，先把输入输出抽象成可测试约束。 | [ml-model-inference-extraction-and-weight-analysis.md](ml-model-inference-extraction-and-weight-analysis.md) |

## 原始资料

- [model-attacks.md](../raw/ai-ml/model-attacks.md)
