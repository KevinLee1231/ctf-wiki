---
type: technique
tags: [reverse, pwn, z3, symbolic-execution, constraints]
skills: [ctf-reverse, ctf-pwn]
raw:
  - ../raw/pwn/vm-z3-sandbox-and-game-basics.md
  - ../raw/reverse/vm-obfuscation-transform-patterns.md
updated: 2026-07-27
---

# Constraint Solver and Symbolic-State Recovery

## 适用场景

校验、VM、递推或游戏状态可表示为有限位向量、整数、数组或路径条件；手工逆运算困难但状态规模可控，适合精确建模后交给 SMT/符号执行器。

## 识别信号

- 多轮位运算、条件分支、索引约束或 checksum 同时约束输入。
- 题目给出可判定验证器，候选可快速复算。
- 路径数量可通过固定分支、分块或 concrete execution 限制。

## 最小证据

- 明确每个变量的位宽、符号、溢出和数组边界。
- 模型在已知输入上与原程序逐步状态一致。
- Solver 输出经原验证器确认，而不只满足简化模型。

## 解法骨架

1. 先消除常量、死分支和可直接逆运算部分。
2. 按真实位宽建立 BitVec/Int/Array，逐轮对齐中间状态。
3. 用分块、增量求解、路径约束或已知格式减少搜索。
4. 枚举/排除多解，并用原程序端到端验证。

## 关键变体

- 位向量校验和多轮变换。
- VM/bytecode 符号提升。
- 状态机、递推、路径条件与混合 concrete-symbolic 执行。

## 常见陷阱

- 用无界整数模拟机器溢出。
- 约束过少得到伪解，约束过多把未验证假设写入模型。
- 一次性符号化全部循环导致状态爆炸。

## 关联技巧

- [vm-z3-sandbox-and-game-basics.md](vm-z3-sandbox-and-game-basics.md)
- [custom-vm-and-wasm-state-lifting.md](custom-vm-and-wasm-state-lifting.md)
- [linear-prng-state-and-seed-recovery.md](linear-prng-state-and-seed-recovery.md)

## 原始资料

- [vm-z3-sandbox-and-game-basics.md](../raw/pwn/vm-z3-sandbox-and-game-basics.md)
- [vm-obfuscation-transform-patterns.md](../raw/reverse/vm-obfuscation-transform-patterns.md)
