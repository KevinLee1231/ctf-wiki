---
type: technique
tags: [reverse, binary-diff, function-matching, symbol-migration, bindiff, diaphora]
skills: [ctf-reverse, ctf-pwn]
raw:
  - ../raw/reverse/packers-deobfuscation-and-debug-automation.md
  - ../raw/reverse/ACTF2023-tree-wp.md
  - ../raw/reverse/D3CTF2023-d3recover-wp.md
updated: 2026-08-03
---

# Binary Diff and Symbol Migration

## 适用场景

已有同源 old/new、patched/unpatched、debug/release、不同平台或一份已分析样本，需要用它缩小另一份 binary 的搜索范围、定位语义变化，或迁移已验证的符号/注释/类型。

## 识别信号

- 题目同时给出两份同源 binary、补丁、升级版本、服务端/客户端或带符号参考版。
- 编译器、地址、布局或内联发生变化，但 exports、字符串、常量、调用图邻域或外部 I/O 仍保留锚点。
- 直接按地址对齐已失效，却能通过函数形状和 caller/callee 关系找到候选。

## 最小证据

- 记录两份输入的 hash、格式、架构、编译/保护差异和加载基址，保证比较对象确实同源。
- 每个候选匹配至少有两类独立锚点：exports/strings/constants、CFG/size、call-graph neighborhood、已知 I/O 或独特副作用。
- 符号迁移前保留“源函数→目标函数→匹配理由→验证结果”的映射表。

## 解法骨架

1. 固定比较边界：同一架构/不同架构、同一编译器/不同优化、是否脱壳，以及是要找变化还是要迁移知识。
2. 用 exports、稳定字符串、独特常量和已知入口建立高置信 seed matches。
3. 从 seed 向 caller/callee 邻域扩展，综合函数大小、基本块/边数、循环、调用集和常量建立候选；不使用单一相似度阈值盲目批量重命名。
4. 对每个将要迁移的函数验证 xref、调用目标、参数/返回值、关键数据结构和已知输入输出。
5. 先迁移函数名和注释，后迁移类型/局部变量；结构布局改变时必须重新核对 field offset。
6. 在第一个真正语义变化处停止“迁移”，改为独立分析 change semantics；不要用旧名称遮蔽新逻辑。

## 工具选择

| 范围 | 选择 |
|---|---|
| 小样本/少量函数 | `radiff2`、导出表/字符串差分、IDA/Ghidra 手工并排 |
| 大量函数/长期分析迁移 | BinDiff、Diaphora 或其他可导出匹配依据的图差分工具 |
| 跨架构/优化差异较大 | 弱化 raw bytes 权重，增加调用图、常量、语义副作用和 I/O 验证 |

工具产生的 match 只是候选证据。LLM 可以帮助解释匹配理由或排序候选，但不应直接把指令地址当成 call target、也不应未验证就批量写回名称/类型。

## 路由边界

- 目标是定位逻辑变化、恢复算法或迁移符号：留在 `ctf-reverse`。
- 差分已经确认可利用的内存破坏、竞态或内核 primitive，后续目标是 PoC/exploit：转 `ctf-pwn`。
- 补丁差分不自动意味可武器化；必须独立证明触发条件、可达性和安全影响。

## 常见陷阱

- 不同编译优化、LTO、函数内联/拆分或脱壳状态产生大量伪差异。
- 候选匹配仅因大小相似就被接受，或调用图邻居本身也是错配。
- 从旧版迁移 struct/type 后没有重新核对 offset，伪代码因此看似合理但数据流错位。
- 把差分工具的 confidence 当成语义相等证明。

## 关联技巧

- [packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md)
- [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md)
- [known-cves-and-n-day-exploits.md](known-cves-and-n-day-exploits.md)

## 原始资料

- [packers-deobfuscation-and-debug-automation.md](../raw/reverse/packers-deobfuscation-and-debug-automation.md)
- [ACTF2023-tree-wp](../raw/reverse/ACTF2023-tree-wp.md)
- [D3CTF2023-d3recover-wp](../raw/reverse/D3CTF2023-d3recover-wp.md)
