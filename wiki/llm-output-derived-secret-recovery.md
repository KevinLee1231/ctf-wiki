---
type: technique
tags: [ai-ml, llm, weak-randomness, key-derivation, sampling]
skills: [ctf-ai-ml, ctf-crypto]
raw:
  - ../raw/ai-ml/SUCTF2026-easyLLMWP.md
  - ../raw/ai-ml/llm-attacks.md
updated: 2026-07-27
---

# LLM Output-Derived Secret Recovery

## 适用场景

应用把可复现或低熵的 LLM 文本直接哈希、截断或编码成 key、seed、password 或 token，且存在密文、padding、格式或登录反馈可验证候选。

## 识别信号

- 模型、prompt、temperature、采样参数和派生函数公开或可推断。
- 低温、固定模板或短输出使候选分布高度集中。
- 派生秘密存在离线验证器，例如 AES padding、文件头或已知明文格式。

## 最小证据

- 精确复现模型版本、聊天模板、prompt 和输出后处理。
- 证明输出分布显著小于秘密名义空间。
- 对候选派生值有确定 oracle，避免凭“像明文”主观判断。

## 解法骨架

1. 复刻完整生成参数，保留模型原始输出，包括换行和空格。
2. 多批次采样并按频率去重，优先测试高概率候选。
3. 严格按服务端编码、哈希和截断顺序派生秘密。
4. 用 padding、magic、MAC 或协议响应批量验证，命中后重新端到端复算。

## 关键变体

- 固定输出：greedy/temperature 近零时可能一次复现。
- 集中分布：需要按经验频率枚举若干候选。
- 上下文依赖：system prompt、历史消息或模型量化版本会改变分布。

## 常见陷阱

- 忽略尾随换行、Unicode 正规化或字符串编码。
- 使用相近但不同的模型或 chat template。
- 没有强校验 oracle，错误候选被误当成功。

## 关联技巧

- [llm-attacks.md](llm-attacks.md)
- [prompt-injection-and-tool-call-hijacking.md](prompt-injection-and-tool-call-hijacking.md)
- [token-smuggling-and-output-constraint-bypass.md](token-smuggling-and-output-constraint-bypass.md)

## 原始资料

- [SUCTF2026-easyLLMWP](../raw/ai-ml/SUCTF2026-easyLLMWP.md)
- [llm-attacks.md](../raw/ai-ml/llm-attacks.md)
