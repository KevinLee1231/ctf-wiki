---
type: technique
tags: [pwn, stack-overflow, rop, stack-pivot, control-flow]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/overflow-basics.md
  - ../raw/pwn/stack-pivots-srop-and-seccomp-rop.md
  - ../raw/pwn/LilacCTF2026-gate-way-wp.md
updated: 2026-07-28
---

# Stack Control-Flow and Constrained ROP

## 适用场景

栈溢出、栈迁移或有限覆盖可控制返回地址，但 gadget、输入长度、坏字符、寄存器状态或 seccomp 限制使常规 ret2libc 不可直接使用。

## 识别信号

- 可覆盖 saved return address、saved frame pointer 或异常返回上下文。
- 可写区、二次 read、leave/ret、pop gadget 或可控调用序列可用。
- NX/PIE/canary/ASLR 或输入字符约束决定链长和落点。

## 最小证据

- 用 cyclic/调试器确认精确 offset 和可控字节。
- 列出保护、泄露能力、可写内存、gadget 和 syscall 约束。
- 最小链能稳定控制 PC，并保持栈对齐。

## 解法骨架

1. 先构造单次可观察控制流转移。
2. 若原栈空间不足，将第二阶段读入 `.bss`/heap 并迁移栈。
3. 按现有泄露和保护选择 ret2libc、ORW、SROP 或 constrained ROP。
4. 对每个 gadget 记录寄存器副作用、栈消耗和 ABI 对齐，端到端复验。

## 关键变体

- Partial overwrite：利用地址低字节和低熵映射。
- Stack pivot：`leave; ret`、可控 `rsp` 或调用帧迁移。
- Constrained ROP：坏字符、短输入、CET/seccomp 下组合有限 gadget。

## 常见陷阱

- 只看 gadget 语义，忽略额外 pop/内存写副作用。
- x86-64 调用前栈未按 ABI 对齐。
- 本地 libc/loader 与远端不同，地址链不可迁移。

## 关联技巧

- [overflow-basics.md](overflow-basics.md)
- [stack-pivots-srop-and-seccomp-rop.md](stack-pivots-srop-and-seccomp-rop.md)
- [dynamic-linker-and-symbol-resolution-exploitation.md](dynamic-linker-and-symbol-resolution-exploitation.md)

## 原始资料

- [overflow-basics.md](../raw/pwn/overflow-basics.md)
- [stack-pivots-srop-and-seccomp-rop.md](../raw/pwn/stack-pivots-srop-and-seccomp-rop.md)
- [LilacCTF2026-gate-way-wp](../raw/pwn/LilacCTF2026-gate-way-wp.md)
