---
type: family
tags: [ai-ml, family, llm, prompt-injection, jailbreak, tool-use]
skills: [ctf-ai-ml]
raw:
  - ../raw/ai-ml/llm-attacks.md
  - ../raw/ai-ml/SUCTF2026-easyLLMWP.md
  - ../raw/ai-ml/RCTF2025-the-alchemists-cage-wp.md
  - ../raw/pentest/WMCTF2025-shopping-company-phishing-email-wp.md
  - ../raw/ai-ml/VNCTF2026-huntingagent-wp.md
updated: 2026-07-27
---

# LLM Attacks

## 作用边界

本页是 LLM prompt injection、jailbreak、token smuggling、context manipulation 和 tool-use exploitation family。判断重点不是“模型说了什么”，而是模型输出、上下文、工具调用和外部执行器之间的信任边界。

## 识别信号

- 题目有聊天机器人、agent、客服 AI、工具调用、文件分析、检索增强、隐藏提示词或输出格式约束。
- 成功条件是泄露系统提示、历史对话、secret，或诱导模型调用工具读取/解压/执行附件。
- 输入可能来自用户 prompt，也可能来自网页、邮件、文档、附件、检索结果或上下文窗口。

## 最小证据

- 模型可见上下文、角色边界、拒答规则、输出格式和可调用工具列表。
- 工具调用的参数、文件类型、执行器、权限和返回内容。
- 至少一个可复验 prompt，能稳定改变模型行为或工具调用路径。
- 如果目标是 secret 泄露，要确认泄露的是受保护内容而非模型幻觉。

## 路由表

| 证据 | 先验证 | 下一跳 |
|---|---|---|
| 直接 prompt injection | 角色/规则是否可覆盖，是否能复述隐藏内容 | 最小对话脚本 |
| 间接 prompt injection | 外部文档/网页/邮件是否进入上下文 | 控制载体内容并观察引用 |
| jailbreak | 拒答边界、分段输出、角色扮演和编码是否影响策略 | 记录稳定变体 |
| token smuggling | tokenizer、不可见字符、分隔符和编码差异 | 保留原始字节 |
| context window manipulation | 长上下文覆盖、截断、摘要和优先级 | 控制关键指令位置 |
| tool use exploitation | 工具参数是否可被 prompt 控制 | 审计文件读、解压、执行和网络边界 |
| LLM 输出作为 key/seed/password | 模型、prompt、temperature 和输出格式是否已知，输出分布是否集中 | 批量采样候选输出，派生 key 后用密文/校验 oracle 碰撞验证 |
| agent 附件链 | 附件是否能从“分析”推进到执行 | 转 [scripts-and-obfuscation.md](scripts-and-obfuscation.md) 或 malware |

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| 不可信 prompt、网页、邮件或附件能改变 tool call | [prompt-injection-and-tool-call-hijacking.md](prompt-injection-and-tool-call-hijacking.md) |
| Unicode、token 分片、编码或输出后处理产生表示差异 | [token-smuggling-and-output-constraint-bypass.md](token-smuggling-and-output-constraint-bypass.md) |
| 模型输出被直接派生为 key、seed、password 或 token | [llm-output-derived-secret-recovery.md](llm-output-derived-secret-recovery.md) |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [RCTF2025-the-alchemists-cage-wp](../raw/ai-ml/RCTF2025-the-alchemists-cage-wp.md) | 隐藏提示词泄露题的最小证据是模型能否复述历史对话、系统提示或被要求保护的 secret。 |
| [SUCTF2026-easyLLMWP](../raw/ai-ml/SUCTF2026-easyLLMWP.md) | 服务端公开 `key = SHA256(LLM_output)[:16]`、模型、prompt 和 temperature；低温输出空间集中时，把 LLM 当弱 key generator，批量采样候选输出后用 AES-CBC 密文和 padding/明文格式 oracle 碰撞。 |
| [VNCTF2026-huntingagent-wp](../raw/ai-ml/VNCTF2026-huntingagent-wp.md) | Multi-Agent 审计平台要同时看 Supervisor 截断审查、Coordinator ReAct 调度和 Skill 触发概率；prompt leak 与工具执行可各拿一段 secret。 |
| [WMCTF2025-shopping-company-phishing-email-wp](../raw/pentest/WMCTF2025-shopping-company-phishing-email-wp.md) | 客服 AI 会分析附件时，prompt injection 可以把“解压并检查 zip”推进到工具执行 ELF；判断重点是工具链而不是聊天输出本身。 |
| [ACTF2023-slm-wp](../raw/ai-ml/ACTF2023-slm-wp.md) | 本题的决定性问题不是传统意义上的“让模型说出秘密”，而是应用把模型生成的文本直接送入 Python 执行器。审计 LLM 应用时，应沿数据流检查模型输出最终进入了普通文本、结构化解析器，还是 `exec`、`eval`、解释器或系统命令等执行边界。 |
| [WMCTF2024-give-your-shell-wp](../raw/ai-ml/WMCTF2024-give-your-shell-wp.md) | 第一层：prompt 中直接泄露 flag；第二层：工具函数黑名单过窄，`os.popen` 是真实命令执行点。 |

## 合并与拆分结论

- 保留为 family：LLM 攻击覆盖 prompt、上下文、工具和附件链，多数题需要先判断边界。
- 不合并进 adversarial ML：LLM 攻击不依赖梯度/扰动范数，验证方式不同。
- prompt/tool、token/输出约束和输出派生秘密已拆为三个 technique；本页只保留三者之间的首轮边界。

## 常见误判

- 只看聊天输出，忽略工具调用才是执行边界。
- 把模型编造的 secret 当作成功，未做外部验证。
- 间接注入没有确认恶意文本确实进入模型上下文。
- 工具调用参数被日志或 schema 修正，实际没有执行攻击者意图。

## 关联页面

- [adversarial-ml.md](adversarial-ml.md)
- [ml-model-inference-extraction-and-weight-analysis.md](ml-model-inference-extraction-and-weight-analysis.md)
- [block-mode-misuse-family.md](block-mode-misuse-family.md)
- [scripts-and-obfuscation.md](scripts-and-obfuscation.md)
- [ai-ml-tooling.md](ai-ml-tooling.md)

## 原始资料

- [llm-attacks.md](../raw/ai-ml/llm-attacks.md)
- [SUCTF2026-easyLLMWP.md](../raw/ai-ml/SUCTF2026-easyLLMWP.md)
- [RCTF2025-the-alchemists-cage-wp.md](../raw/ai-ml/RCTF2025-the-alchemists-cage-wp.md)
- [WMCTF2025-shopping-company-phishing-email-wp](../raw/pentest/WMCTF2025-shopping-company-phishing-email-wp.md)
- [VNCTF2026-huntingagent-wp](../raw/ai-ml/VNCTF2026-huntingagent-wp.md)
