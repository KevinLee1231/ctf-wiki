---
type: family
tags: [ai-ml, model-extraction, inference, weight-analysis, family]
skills: [ctf-ai-ml]
raw:
  - ../raw/ai-ml/model-attacks.md
  - ../raw/ai-ml/LilacCTF2026-residue-wp.md
  - ../raw/ai-ml/SU_谁是小偷WP.md
  - ../raw/ai-ml/SU_我不是神偷WP.md
  - ../raw/ai-ml/SU_babyAIWP.md
  - ../raw/ai-ml/SU_easyLLMWP.md
  - ../raw/ai-ml/SU_theifWP.md
updated: 2026-07-06
---

# ML Model Inference, Extraction and Weight Analysis

## 作用边界

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

## 分流流程

1. 先做 I/O 契约：输入类型、shape、归一化、tokenizer、输出字段和限制次数。
2. 如果能查询服务，构造少量代表性查询样本，观察标签、概率、长度、错误和边界反馈。
3. 如果有权重或 checkpoint，检查 config、state_dict、异常层、adapter 合并和权重差分。
4. 根据反馈选择路线：抽取近似模型、反演输入、搜索碰撞、恢复隐藏类别、推断训练样本或合并/抵消权重扰动。
5. 输出可复验样本或脚本，证明模型产生目标分类、目标文本、目标 embedding 或恢复出的 flag。

## 模型路线分流

| 模型路线 | 首轮判断 |
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

- [linear-model-parameter-recovery.md](linear-model-parameter-recovery.md)
- [linear-model-input-lattice-recovery.md](linear-model-input-lattice-recovery.md)
- [transformer-logit-inversion.md](transformer-logit-inversion.md)
- [adversarial-ml.md](adversarial-ml.md)
- [llm-attacks.md](llm-attacks.md)
- [ai-ml-tooling.md](ai-ml-tooling.md)

## WP 案例沉淀

本节只抽取 raw WP 中可复用的识别信号和下一跳，不替代原始题解正文。

| Raw WP | 复用信号 | 下一跳 |
|---|---|---|
| [LilacCTF2026-residue-wp](../raw/ai-ml/LilacCTF2026-residue-wp.md) | 已知 GPT-2 Medium 权重和目标 logits，可逐位置枚举词表并用 MSE 匹配 logits；KV cache 是可接受复杂度的关键。 | [transformer-logit-inversion.md](transformer-logit-inversion.md) |
| [SU_谁是小偷WP](../raw/ai-ml/SU_谁是小偷WP.md) | `Conv2d -> Flatten -> Linear` 且无激活函数，`/predict` 返回完整向量；先用 basis query 恢复整体仿射映射，再拆参数。 | [linear-model-parameter-recovery.md](linear-model-parameter-recovery.md) |
| [SU_我不是神偷WP](../raw/ai-ml/SU_我不是神偷WP.md) | 附件结构与线上形状冲突，`/flag` 报错暴露真实 state_dict；先恢复共享线性层，再把两层卷积分解为等效核。 | [linear-model-parameter-recovery.md](linear-model-parameter-recovery.md) |
| [SU_babyAIWP](../raw/ai-ml/SU_babyAIWP.md) | `Conv1d -> Linear` 无激活且权重藏在 `model.pth`；展开成带小噪声的模线性方程后用 LLL/Babai 恢复 flag 字节。 | [linear-model-input-lattice-recovery.md](linear-model-input-lattice-recovery.md) |
| [SU_easyLLMWP](../raw/ai-ml/SU_easyLLMWP.md) | LLM 输出被直接派生为 AES key，且模型、prompt、temperature 和输出格式已知；先采样候选输出并用密文/PKCS#7 oracle 碰撞验证，不要当普通 prompt injection 处理。 | [llm-attacks.md](llm-attacks.md)、[block-mode-misuse-family.md](block-mode-misuse-family.md) |
| [SU_theifWP](../raw/ai-ml/SU_theifWP.md) | 模型上传接口只比较部分参数，四维卷积权重校验缺口可配合 `/predict` 输出恢复被检查线性层。 | [linear-model-parameter-recovery.md](linear-model-parameter-recovery.md) |

## 原始资料

- [model-attacks.md](../raw/ai-ml/model-attacks.md)
- [LilacCTF2026-residue-wp](../raw/ai-ml/LilacCTF2026-residue-wp.md)
- [SU_谁是小偷WP](../raw/ai-ml/SU_谁是小偷WP.md)
- [SU_我不是神偷WP](../raw/ai-ml/SU_我不是神偷WP.md)
- [SU_babyAIWP](../raw/ai-ml/SU_babyAIWP.md)
- [SU_easyLLMWP](../raw/ai-ml/SU_easyLLMWP.md)
- [SU_theifWP](../raw/ai-ml/SU_theifWP.md)
