---
type: technique
tags: [crypto, prng, mt19937, lcg, seed, state-recovery]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/mt-lcg-and-seed-recovery.md
  - ../raw/crypto/prng-z3-lcg-and-timing-attacks.md
updated: 2026-07-27
---

# Linear PRNG State and Seed Recovery

## 适用场景

随机数来自 MT19937、LCG、语言运行时 PRNG 或可枚举时间种子，输出足以恢复内部状态、参数或初始 seed，并继续预测 token、key 或游戏结果。

## 识别信号

- 连续暴露大量整数、截断输出或由 PRNG 派生的 token。
- 递推形如 `x[n+1]=a*x[n]+c mod m`，或输出宽度符合 MT/语言运行时。
- seed 依赖秒级时间、PID、启动顺序或小范围业务字段。

## 最小证据

- 确认输出调用顺序、位宽、符号、截断和跳过次数。
- LCG 至少有足够相邻状态或差分关系；MT 需评估可见总比特是否覆盖状态。
- 时间种子必须给出可审计窗口，而不是无限枚举。

## 解法骨架

1. 复刻目标运行时的 PRNG 与输出转换。
2. 完整输出优先反 temper/解线性方程；LCG 用差分 GCD 求模数和参数。
3. 截断输出建立格或 SMT 约束；时间 seed 则限定窗口并并行验证。
4. 恢复状态后按真实调用顺序前进，用后续已知输出确认同步。

## 关键变体

- MT state clone：足量完整输出可直接恢复 624-word 状态。
- Truncated LCG：未知低位或高位通常需要格/约束求解。
- Runtime/time seed：JavaScript、C、Python 的转换和调用次数各不相同。

## 常见陷阱

- 状态正确但调用偏移一位，预测全部失败。
- 混淆 `random()` 浮点输出和底层整数状态。
- 忽略多线程、重采样或拒绝采样消耗额外输出。

## 关联技巧

- [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md)
- [prng-z3-lcg-and-timing-attacks.md](prng-z3-lcg-and-timing-attacks.md)
- [lattice-small-root-and-partial-leakage.md](lattice-small-root-and-partial-leakage.md)
- [stream-cipher-keystream-reuse-and-state-recovery.md](stream-cipher-keystream-reuse-and-state-recovery.md)

## 原始资料

- [mt-lcg-and-seed-recovery.md](../raw/crypto/mt-lcg-and-seed-recovery.md)
- [prng-z3-lcg-and-timing-attacks.md](../raw/crypto/prng-z3-lcg-and-timing-attacks.md)
