---
type: technique
tags: [ai-ml, technique, transformer, logits, inversion, token-recovery]
skills: [ctf-ai-ml]
raw:
  - ../raw/ai-ml/LilacCTF2026-residue-wp.md
updated: 2026-07-06
---

# Transformer Logit Inversion

## 适用场景

已知 Transformer/LLM 权重、tokenizer 和某段隐藏输入前向传播后的逐位置 logits，需要从 logits 反推出原始 token 序列。关键不是训练、对抗样本或 prompt 绕过，而是利用固定模型前向传播的确定性做原像搜索。

## 识别信号

- 附件给出 `target_logits.npy`、模型名、tokenizer 或 checkpoint。
- 目标是恢复原始文本、flag token 序列或 prompt，而不是让模型输出某句话。
- 对同一模型和上下文，正确 token 的 logits 与目标几乎数值一致，错误 token 的 MSE 明显更大。
- 题解或论文线索出现 logit inversion、SipIt、teacher logits、distillation artifact。

## 最小证据

- 模型版本、tokenizer、dtype、device 和 logits shape 与出题环境一致。
- 已确认 logits 是逐位置输出，而不是采样后的 token 概率截断或只保留 top-k。
- 至少用前一两个 token 做枚举验证：正确候选的 MSE 应接近 0。

## 解法骨架

1. 加载目标 logits、同版本 tokenizer 和模型权重。
2. 维护已经恢复出的 token 历史。
3. 对当前位置枚举词表候选，把候选接到历史后前向传播。
4. 用 MSE/最大误差比较候选 logits 与目标 logits，选误差最小且接近 0 的 token。
5. 固定 token 后更新 `past_key_values`；枚举下一位时复用 KV cache，避免每轮重算完整历史。
6. 用 tokenizer decode，并正向复算完整 logits 校验恢复结果。

## 关键变体

| 变体 | 判断方式 | 处理 |
|---|---|---|
| 完整 logits | 每个位置保留完整词表 logits | 逐 token 全词表枚举，MSE 直接判定。 |
| top-k / 截断 logits | 只给部分候选或概率 | 先确认信息是否足够；可能需要 beam search 或语言先验。 |
| 浮点误差明显 | 正确 token 误差不接近 0 | 检查模型版本、tokenizer、dtype、position id 和 cache 对齐。 |

## 常见陷阱

- 把模型采样随机性和前向传播确定性混为一谈。
- tokenizer 版本不一致，导致同一文本被切成不同 token。
- 每个候选都重算完整历史，复杂度爆炸。
- logits 维度或位置偏移一位，导致所有候选误差都不可信。

## 关联技巧

- [ml-model-inference-extraction-and-weight-analysis.md](ml-model-inference-extraction-and-weight-analysis.md)
- [llm-attacks.md](llm-attacks.md)
- [ai-ml-tooling.md](ai-ml-tooling.md)

## 原始资料

- [LilacCTF2026-residue-wp](../raw/ai-ml/LilacCTF2026-residue-wp.md)
