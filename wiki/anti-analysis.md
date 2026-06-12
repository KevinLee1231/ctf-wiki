---
type: technique
tags: [reverse, technique]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/anti-analysis.md
updated: 2026-05-21
---

# Anti-Analysis Techniques & Bypasses

## 适用场景

需要理解二进制、脚本、字节码、壳、VM、固件或混淆逻辑，再恢复算法或输入。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 附件是 ELF/PE/APK/WASM/pyc/固件/脚本，或存在壳、SMC、自定义 VM。
- flag 校验藏在运行时生成代码、解密字符串、解释器或 native 扩展中。
- 静态字符串不足，需要交叉引用、动态断点或 trace。
- 题面或 raw 线索能落到这些关键词之一：Linux Anti-Debug (Advanced)、ptrace-Based、/proc Filesystem Checks、Timing-Based Detection、Signal-Based Anti-Debug、Syscall-Level Evasion、Windows Anti-Debug (Advanced)、PEB (Process Environment Block) Checks。

## 最小证据

- 已完成主方向判断，并确认本页技巧比相邻技巧更能解释当前证据。
- 至少有一个可复现输入、输出、文件结构、数学关系、协议行为或运行时状态。
- 能指出 raw 案例中哪一个变体与当前题最接近，以及不同点在哪里。

## 解法骨架

1. 先做载体、字符串、导入和入口函数首检。
2. 定位真实校验、解密、分发或比较点。
3. 把复杂逻辑降维成约束、解密脚本或 oracle。
4. 用 solver / forward check 验证输入。

## 关键变体

| 变体 | 复用重点 |
|---|---|
| Linux Anti-Debug (Advanced) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| ptrace-Based | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| /proc Filesystem Checks | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Timing-Based Detection | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Signal-Based Anti-Debug | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Syscall-Level Evasion | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Windows Anti-Debug (Advanced) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| PEB (Process Environment Block) Checks | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| NtQueryInformationProcess | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Heap Flags | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| TLS Callbacks | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Hardware Breakpoint Detection | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Software Breakpoint Detection (INT3 Scanning) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Exception-Based Anti-Debug | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| NtSetInformationThread (Thread Hiding) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Anti-VM / Anti-Sandbox | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| CPUID Hypervisor Bit | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| MAC Address / Hardware Fingerprinting | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Timing-Based VM Detection | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| File / Registry Artifacts | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Resource Checks (CPU Count, RAM, Disk) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Anti-DBI (Dynamic Binary Instrumentation) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Frida Detection | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Pin/DynamoRIO Detection | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

## 常见陷阱

- 只按关键词跳页，没有先构造最小证据。
- 照搬 raw 中的一次性 payload，没有检查当前题的边界条件。
- 忽略相邻技巧之间的 pivot，导致在错误方向上继续投入时间。

## 关联技巧

- [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md)
- [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)
- [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md)
- [disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md)
- [embedded-python-pyd-custom-aes.md](embedded-python-pyd-custom-aes.md)
- [font-shader-firmware-and-legacy-patterns.md](font-shader-firmware-and-legacy-patterns.md)

## 原始资料

- [anti-analysis.md](../raw/reverse/anti-analysis.md)
