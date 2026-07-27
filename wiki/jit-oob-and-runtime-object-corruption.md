---
type: technique
tags: [pwn, jit, oob, runtime, object-corruption]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/interpreter-jit-canary-and-integer-exploits.md
  - ../raw/pwn/oob-jit-parser-primitives.md
updated: 2026-07-27
---

# JIT OOB and Runtime-Object Corruption

## 适用场景

JIT 优化假设、数组边界、类型反馈或解释器对象布局错误产生 OOB 读写，需要从浮点/Tagged value 转换建立 addrof/fakeobj，再推进到任意读写和可执行目标。

## 识别信号

- 热身/优化后行为改变，禁用 JIT 或改变类型反馈后漏洞消失。
- 数组 length、elements pointer、map/shape 或 boxed value 可被越界影响。
- 值以 NaN-boxing、SMI、compressed pointer 等形式编码。

## 最小证据

- 固定引擎 commit/build flags，并保存稳定触发脚本。
- 还原对象 header、元素存储和 pointer compression 规则。
- 分阶段验证 OOB、addrof/fakeobj 和任意读写。

## 解法骨架

1. 最小化优化触发条件和类型反馈序列。
2. 用相邻数组/对象布局把 OOB 转成地址泄露或长度扩大。
3. 构造 addrof/fakeobj，再搭建受控 backing-store 任意读写。
4. 根据 W^X、sandbox 和 pointer cage 选择 WASM、JIT code、ROP 或跨隔离区目标。

## 关键变体

- Bounds-check elimination。
- Type confusion / map transition。
- Pointer compression 与 engine sandbox escape。

## 常见陷阱

- GC/对象布局未固定，利用仅偶尔成功。
- 浮点与 64 位整数转换丢失 NaN payload。
- 只在 debug build 成功，release 优化路径不同。

## 关联技巧

- [interpreter-jit-canary-and-integer-exploits.md](interpreter-jit-canary-and-integer-exploits.md)
- [oob-jit-parser-primitives-family.md](oob-jit-parser-primitives-family.md)
- [data-interpretation-memory-primitives.md](data-interpretation-memory-primitives.md)

## 原始资料

- [interpreter-jit-canary-and-integer-exploits.md](../raw/pwn/interpreter-jit-canary-and-integer-exploits.md)
- [oob-jit-parser-primitives.md](../raw/pwn/oob-jit-parser-primitives.md)
