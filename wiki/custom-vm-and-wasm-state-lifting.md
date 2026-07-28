---
type: technique
tags: [reverse, vm, wasm, lifting, state-machine]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/vm-obfuscation-transform-patterns.md
  - ../raw/web/game-state-websocket-and-wasm.md
  - ../raw/reverse/ACTF2026-virtualnpu-wp.md
updated: 2026-07-28
---

# Custom VM and WASM State Lifting

## 适用场景

目标逻辑运行在自定义 VM、字节码解释器或 WASM linear memory 中；需要恢复 instruction semantics、state layout 和控制流，再提升到可求解的中间表示。

## 识别信号

- Dispatch loop、opcode table、虚拟寄存器/栈或线性内存反复更新。
- 主程序只加载 bytecode/module，真实校验位于解释器。
- WASM exports/imports、memory offset 或 table indirect call 关联关键状态。

## 最小证据

- 列出 opcode 编码、操作数宽度、PC 更新和状态结构。
- 对每条已恢复指令用短程序验证语义。
- 解释器执行与自写 emulator/lifter 的 state trace 一致。

## 解法骨架

1. 定位 fetch-decode-execute 和 state 初始化。
2. 逐 opcode 记录读写集合、分支和副作用。
3. 将 bytecode 提升为 CFG/SSA/约束，或编写 emulator。
4. 用 trace 对齐后再反解输入、patch 状态或符号执行。

## 关键变体

- Stack/register VM。
- Flattened/virtualized native VM。
- WASM linear memory、imports/exports 与 host bridge。

## 常见陷阱

- 只命名 opcode，没有验证 PC/flag/异常语义。
- 混淆 bytecode 端序和变长操作数。
- 忽略 host import 对状态的外部影响。

## 关联技巧

- [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md)
- [game-state-websocket-and-wasm.md](game-state-websocket-and-wasm.md)
- [constraint-solver-and-symbolic-state-recovery.md](constraint-solver-and-symbolic-state-recovery.md)

## 原始资料

- [vm-obfuscation-transform-patterns.md](../raw/reverse/vm-obfuscation-transform-patterns.md)
- [game-state-websocket-and-wasm.md](../raw/web/game-state-websocket-and-wasm.md)
- [ACTF2026-virtualnpu-wp](../raw/reverse/ACTF2026-virtualnpu-wp.md)
