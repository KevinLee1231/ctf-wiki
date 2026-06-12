---
type: tooling
tags: [reverse, tooling, emulation, tracing, instrumentation]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/qiling-triton-pin-and-ldpreload.md
updated: 2026-06-11
---

# Qiling, Triton, Pin and LD_PRELOAD

## 作用边界

本页覆盖逆向中的重型运行控制工具：完整环境模拟、指令级动态符号执行、二进制插桩计数和动态库劫持。它们适合在普通调试器、反汇编器或 Frida hook 已经无法稳定观察目标时使用，但使用成本更高，必须先明确要控制的环境变量、系统调用、输入长度、目标分支或侧信道指标。

## 工具路由

| 工具 | 适合的问题 | 最小前置条件 | 典型失败信号 |
|---|---|---|---|
| Qiling | 需要在可控 rootfs 中跑 ELF/PE/shellcode，hook syscall/API，绕过环境/反调试，或做输入 fuzz | 已知架构、入口、依赖文件和目标函数；能构造最小 rootfs 或隔离执行片段 | syscall/API 缺失、rootfs 不匹配、库版本不对、路径/权限与原程序环境不同。 |
| Triton | 想从真实执行中抽取指令级符号表达式，恢复输入约束或做污点分析 | 已能跑到目标代码；输入长度和符号化字节范围可控 | 表达式过大、符号化范围过宽、间接跳转/库函数导致状态爆炸。 |
| Intel Pin | 通过指令数、basic block、内存访问或 trace 差异恢复输入 | 有可重复运行的本地目标；输入对计数或 trace 有稳定影响 | 计数噪声大、ASLR/线程/时间导致不稳定、差异不随输入单调变化。 |
| LD_PRELOAD | Linux 动态链接目标可被库劫持，需要冻结时间、替换 `memcmp`/`strcmp`/随机数/API | 目标不是静态链接或 setuid 限制；能确认被劫持函数确实参与校验 | hook 不生效、函数被内联、静态链接、符号版本不匹配、程序走 syscall 绕过 libc。 |

## 路线选择

- 目标依赖系统调用、文件路径、环境变量或反调试结果：先考虑 Qiling 或 LD_PRELOAD。
- 目标分支清晰但输入搜索空间过大：先考虑 Triton 或 angr；Triton 更贴近真实指令，angr 更适合路径搜索。
- 目标像黑盒比较器，输入越接近正确值执行越长：用 Pin 或 LD_PRELOAD 建 side channel，再逐字节验证。
- 只需要替换少量函数返回值：优先 Frida 或 LD_PRELOAD；不必先上完整模拟。
- 需要重放某段代码但不想搭完整进程：先尝试 Unicorn；只有依赖系统环境时再升级到 Qiling。

## 常见失败与修正

- Qiling 跑不起来时，先把目标缩到单函数或最小 syscall 集合；不要一开始完整复刻系统。
- Triton 表达式爆炸时，减少符号输入、concretize 无关内存、hook 库函数。
- Pin 计数不稳定时，固定 ASLR、输入长度、线程和随机源，多次采样确认差异是否可复现。
- LD_PRELOAD 无效时，检查 `ldd`、静态链接、setuid、符号版本和目标是否直接 syscall。
- 侧信道路线每一步都要做 forward check；不要只相信计数最高或路径最短的候选字节。

## 关联页面

- [disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md)
- [frida-angr-lldb-and-x64dbg.md](frida-angr-lldb-and-x64dbg.md)
- [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md)
- [packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md)
- [anti-analysis.md](anti-analysis.md)
- [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md)

## 原始资料

- [qiling-triton-pin-and-ldpreload.md](../raw/reverse/qiling-triton-pin-and-ldpreload.md)
