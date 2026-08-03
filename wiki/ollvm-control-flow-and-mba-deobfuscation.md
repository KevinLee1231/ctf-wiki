---
type: technique
tags: [reverse, ollvm, control-flow-flattening, opaque-predicate, mba, deobfuscation]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/anti-analysis.md
  - ../raw/reverse/ACTF2023-obfuse-wp.md
  - ../raw/reverse/ACTF2025-unstoppable-wp.md
updated: 2026-08-03
---

# OLLVM Control-Flow and MBA Deobfuscation

## 适用场景

伪代码被 dispatcher/state variable、opaque predicate、bogus branch、instruction substitution 或 mixed Boolean-arithmetic（MBA）表达式主导。本页适用于 OLLVM 或 OLLVM-like 变体；某个 CFG 形态只是信号，不足以确认具体 pass 或实现来源。

## 分层识别

| 层 | 稳定信号 | 优先产物 |
|---|---|---|
| Control-flow flattening | 中心 dispatcher、state 反复更新、原基本块变成 case/indirect branch | state 转移表、恢复后 CFG |
| Bogus control flow | 恒真/恒假条件、重复块、不可达副作用 | predicate 证明、可达边集 |
| Instruction substitution / MBA | 简单运算被多层 bitwise/arithmetic 恒等式替换 | 定宽语义下的简化表达式 |
| Runtime constants/strings | 全局常量或字符串只在初始化后出现 | 初始化后 dump、实际常量表 |

## 最小证据

- 选一个输入可控、输出可观察的小函数作 ground truth，保留原 CFG、trace 和函数输出。
- 对 flattening 记录 state 的初值、更新点、case 目标和出口；对 MBA 固定 bit width 与 signedness。
- 每消除一层后，简化 CFG 与原程序在已知输入上的可达块、内存副作用和输出一致。

## 解法骨架

1. 先找真实校验或解密边界，不对全程序盲目反混淆。初始化会解出常量时，先 dump 真实数据。
2. 确定主层是 flattening、bogus control flow、MBA 还是组合，并为一个小函数建立未修改 trace。
3. 恢复 dispatcher/state 转移或证明 opaque predicate；每次只重写一类边，不同时大量 patch CFG 和数据流。
4. 把 MBA 提升到明确位宽的 IR/SMT 表达式，做常量传播、恒等式简化和死代码删除；用随机向量只做粗测，最终还需穷尽小域或 SMT/原程序验证。
5. 将简化后结果回填为注释、新函数或独立 solver，而不是无验证地覆盖原 binary。
6. 用 ground-truth 输入、分支 trace、关键内存和最终 I/O 四类证据验证，再扩大到其他函数。

## 工具边界

- IDA microcode plugin、Miasm/IR、angr/Z3 和 Unicorn 可以分别服务于伪代码级规则、IR/符号简化、路径证明和语义回放；选择应由当前保护层决定。
- 本机已有的 angr/Z3/Unicorn 足以做小范围验证；专用反混淆插件是按题补充，不是默认前置。实时工具状态见 [reverse-tooling.md](reverse-tooling.md)。
- 不将未知样本上传到公开去混淆服务；题目附件、未发布样本和带凭据数据默认留在本地。

## 常见陷阱

- 路径数量大不等于某个具体 OLLVM 变体；先用 state、CFG 和代码生成特征证明。
- 只修复 CFG，忽略 state 更新中的数据副作用。
- 忽略 C/C++ 未定义行为、整型溢出、位宽和 signedness，把数学恒等式误当成机器语义恒等式。
- 自动脚本产生“更好看”的伪代码后就停止，没有用 trace/I/O 验证。

## 关联技巧

- [packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md)
- [anti-analysis.md](anti-analysis.md)
- [custom-vm-and-wasm-state-lifting.md](custom-vm-and-wasm-state-lifting.md)
- [constraint-solver-and-symbolic-state-recovery.md](constraint-solver-and-symbolic-state-recovery.md)

## 原始资料

- [anti-analysis.md](../raw/reverse/anti-analysis.md)
- [ACTF2023-obfuse-wp](../raw/reverse/ACTF2023-obfuse-wp.md)
- [ACTF2025-unstoppable-wp](../raw/reverse/ACTF2025-unstoppable-wp.md)
