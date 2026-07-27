---
type: technique
tags: [ai-ml, llm, prompt-injection, agent, tool-use]
skills: [ctf-ai-ml, ctf-pentest]
raw:
  - ../raw/ai-ml/llm-attacks.md
  - ../raw/ai-ml/VNCTF2026-huntingagent-wp.md
  - ../raw/pentest/WMCTF2025-shopping-company-phishing-email-wp.md
updated: 2026-07-27
---

# Prompt Injection and Tool-Call Hijacking

## 适用场景

LLM 或 Agent 会把用户输入、邮件、网页、检索结果或附件内容放入上下文，并可进一步调用文件、网络、解释器或命令工具。

## 识别信号

- 外部不可信文本与系统指令进入同一上下文。
- 模型能生成结构化 tool call，工具参数来自模型输出。
- “分析附件”可继续触发解压、读取、执行或联网等高权限动作。

## 最小证据

- 一条可复验输入能稳定改变工具选择或关键参数。
- 保存完整消息角色、工具 schema、调用参数和工具返回。
- 明确秘密或副作用来自真实工具边界，而非模型虚构。

## 解法骨架

1. 沿数据流标出不可信内容进入模型、结构化解析器和工具执行器的每个边界。
2. 用最小指令先验证优先级覆盖，再逐步请求可观察、低副作用的工具动作。
3. 控制附件名、正文、压缩层和引用内容，确认哪一层会进入上下文。
4. 对工具调用做逐步复验，并记录 Supervisor、Coordinator 或截断器是否重写输入。

## 关键变体

- 直接注入：用户 prompt 与系统指令竞争。
- 间接注入：网页、邮件、文档或检索结果携带指令。
- 多 Agent 链：上游模型的摘要、截断或路由决定下游工具是否执行。

## 常见陷阱

- 只追求模型输出敏感文字，忽略真实目标是工具权限。
- 把一次格式解析失败误认为安全边界。
- 未区分模型建议执行与工具实际执行。

## 关联技巧

- [llm-attacks.md](llm-attacks.md)
- [token-smuggling-and-output-constraint-bypass.md](token-smuggling-and-output-constraint-bypass.md)
- [llm-output-derived-secret-recovery.md](llm-output-derived-secret-recovery.md)
- [workflow-runner-internal-api-chain.md](workflow-runner-internal-api-chain.md)

## 原始资料

- [llm-attacks.md](../raw/ai-ml/llm-attacks.md)
- [VNCTF2026-huntingagent-wp](../raw/ai-ml/VNCTF2026-huntingagent-wp.md)
- [WMCTF2025-shopping-company-phishing-email-wp](../raw/pentest/WMCTF2025-shopping-company-phishing-email-wp.md)
