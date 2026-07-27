---
type: technique
tags: [reverse, tracing, hooking, coredump, state-snapshot]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/runtime-patching-oracles-and-tracing.md
  - ../raw/reverse/signal-trace-and-packed-anti-analysis.md
updated: 2026-07-27
---

# Trace, Hook and State-Snapshot Reconstruction

## 适用场景

静态控制流被混淆、动态生成或依赖运行时状态，但关键比较、解密、API 边界或阶段切换可被 hook/trace/coredump 捕获，以事件序列和状态快照反推算法。

## 识别信号

- 关键数据只在短暂内存窗口出现。
- 大量间接跳转/VM/信号处理使静态反编译失真。
- libc/API/比较/解密函数提供稳定观察点。

## 最小证据

- 明确 hook 点、调用约定、输入输出和时间顺序。
- 每个快照记录模块基址、线程、寄存器与内存范围。
- 同一输入重复运行得到可对齐事件。

## 解法骨架

1. 先从高语义 API/比较点记录参数与返回值。
2. 按输入差分定位影响 secret/校验的最小事件片段。
3. 在阶段边界 dump 代码/数据/VM state，并关联 trace 序号。
4. 将观察结果写成独立模拟器，再用新输入交叉验证。

## 关键变体

- API hooking / Frida/LD_PRELOAD。
- 指令/基本块 trace 与父子进程关联。
- Coredump/内存快照恢复短暂明文或状态。

## 常见陷阱

- Trace 全量指令导致噪声失控，没有语义锚点。
- 多线程/ASLR 下事件无法对齐。
- 只复述一次执行轨迹，未抽象成可验证状态转移。

## 关联技巧

- [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md)
- [signal-trace-and-packed-anti-analysis.md](signal-trace-and-packed-anti-analysis.md)
- [anti-debug-self-check-and-environment-bypass.md](anti-debug-self-check-and-environment-bypass.md)

## 原始资料

- [runtime-patching-oracles-and-tracing.md](../raw/reverse/runtime-patching-oracles-and-tracing.md)
- [signal-trace-and-packed-anti-analysis.md](../raw/reverse/signal-trace-and-packed-anti-analysis.md)
