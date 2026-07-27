---
type: technique
tags: [ai-ml, llm, tokenizer, jailbreak, output-filter]
skills: [ctf-ai-ml]
raw:
  - ../raw/ai-ml/llm-attacks.md
  - ../raw/ai-ml/RCTF2025-the-alchemists-cage-wp.md
updated: 2026-07-27
---

# Token Smuggling and Output-Constraint Bypass

## 适用场景

目标依赖拒答策略、关键词过滤、结构化输出或上下文分隔符保护秘密，而模型 tokenizer、规范化器和后处理器对同一字节序列的理解不一致。

## 识别信号

- 空白、Unicode、不可见字符、分段、编码或角色模板会改变拒答结果。
- 模型输出经过 regex、JSON schema、长度限制或敏感词过滤。
- 同一语义换成 token 分片、逐字符输出或中间表示后可越过约束。

## 最小证据

- 保存原始 UTF-8 字节、Unicode code point 和 tokenizer 切分。
- 对照注入前后上下文模板及后处理结果。
- 至少两个语义等价输入表现不同，证明差异来自表示层而非随机性。

## 解法骨架

1. 枚举系统使用的规范化、分隔、tokenization 和输出过滤顺序。
2. 从单一变换开始测试：空白、大小写、Unicode 同形、编码、分片或 JSON 转义。
3. 将目标拆成可通过过滤的中间输出，再在模型外可靠重组。
4. 固定 temperature 和会话状态，多次复验可重复性。

## 关键变体

- 输入侧 smuggling：分隔符或角色模板被 tokenizer 以外的组件重解释。
- 输出侧 bypass：逐字符、索引、编码或结构化字段绕过敏感词检查。
- 上下文窗口操纵：利用截断、摘要或长文本改变高优先级指令的有效位置。

## 常见陷阱

- 只记录渲染后的字符串，丢失决定差异的原始字节。
- 把模型随机回答差异当作 tokenizer 绕过。
- 产出不能确定重组的模糊编码，无法验证真实秘密。

## 关联技巧

- [llm-attacks.md](llm-attacks.md)
- [prompt-injection-and-tool-call-hijacking.md](prompt-injection-and-tool-call-hijacking.md)
- [llm-output-derived-secret-recovery.md](llm-output-derived-secret-recovery.md)

## 原始资料

- [llm-attacks.md](../raw/ai-ml/llm-attacks.md)
- [RCTF2025-the-alchemists-cage-wp](../raw/ai-ml/RCTF2025-the-alchemists-cage-wp.md)
