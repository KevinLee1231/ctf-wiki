---
type: technique
tags: [reverse, vm, wasm, lifting, state-machine]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/vm-obfuscation-transform-patterns.md
  - ../raw/web/game-state-websocket-and-wasm.md
  - ../raw/reverse/ACTF2026-virtualnpu-wp.md
updated: 2026-08-03
---

# Custom VM and WASM State Lifting

## 适用场景

目标逻辑运行在自定义 VM、字节码解释器或 WASM linear memory 中；需要恢复 instruction semantics、state layout 和控制流，再提升到可求解的中间表示。

## 识别信号

- Dispatch loop、opcode table、虚拟寄存器/栈或线性内存反复更新。
- 主程序只加载 bytecode/module，真实校验位于解释器。
- WASM exports/imports、memory offset 或 table indirect call 关联关键状态。
- JavaScript bundle 中的大型常量表、索引器、dispatcher 和 host API bridge 组成软件 VM；具体变量名和 opcode 数值不具备跨样本意义。

## 最小证据

- 列出 opcode 编码、操作数宽度、PC 更新和状态结构。
- 对每条已恢复指令用短程序验证语义。
- 解释器执行与自写 emulator/lifter 的 state trace 一致。

## 解法骨架

1. 定位 fetch-decode-execute、state 初始化和 host bridge；对 JS-hosted VM 同时保留浏览器/network/storage 输入。
2. 逐 opcode 记录操作数宽度、读写集合、PC 变化、分支、异常和外部副作用。
3. 先用少量人工构造的短 bytecode 验证 handler，再将真实 bytecode 提升为 CFG/SSA/约束，或编写 emulator。
4. 对齐原解释器与 lifter/emulator 的逐步 state trace；定位首个分歧的 PC、操作数或 host call。
5. trace 一致后再反解输入、patch 状态或符号执行，并用原解释器做 forward check。

## 关键变体

- Stack/register VM。
- Flattened/virtualized native VM。
- WASM linear memory、imports/exports 与 host bridge。
- JavaScript-hosted VM：minified dispatcher、constant table、worker/WASM 和 browser host API 共同组成状态机。

## 常见陷阱

- 只命名 opcode，没有验证 PC/flag/异常语义。
- 混淆 bytecode 端序和变长操作数。
- 忽略 host import 对状态的外部影响。
- 把一个样本中的变量名、opcode 编号、状态码或固定 offset 当成通用 VM 协议。

## 关联技巧

- [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md)
- [game-state-websocket-and-wasm.md](game-state-websocket-and-wasm.md)
- [constraint-solver-and-symbolic-state-recovery.md](constraint-solver-and-symbolic-state-recovery.md)
- [browser-javascript-runtime-reconstruction.md](browser-javascript-runtime-reconstruction.md)

## 原始资料

- [vm-obfuscation-transform-patterns.md](../raw/reverse/vm-obfuscation-transform-patterns.md)
- [game-state-websocket-and-wasm.md](../raw/web/game-state-websocket-and-wasm.md)
- [ACTF2026-virtualnpu-wp](../raw/reverse/ACTF2026-virtualnpu-wp.md)
