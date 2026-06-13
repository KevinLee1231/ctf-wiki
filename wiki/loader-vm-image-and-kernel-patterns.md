---
type: family
tags: [reverse, family, loader, vm, image-recovery, kernel-module]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/loader-vm-image-and-kernel-patterns.md
updated: 2026-06-12
---

# Loader, VM, Image and Kernel Patterns

## 作用边界

本页是二阶段载体和特殊执行链 family，覆盖 loader、隐藏 opcode、binfmt、shared library backdoor、kernel module maze、image/bitmap 恢复、shellcode 数据段、GBA ROM VM、线程/通道 VM 和无导入/哈希解析样本。它不再作为 technique；它负责把“程序真实逻辑不在第一眼看到的入口”这类证据分流出去。

## 首轮路由

| 信号 | 先确认 | 下一跳 |
|---|---|---|
| 二阶段 loader、`mmap RWX`、数据段 shellcode、execve 递归 | 第一阶段做了什么解密/映射/重启，真实代码何时出现 | [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md)、[packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md) |
| 自定义 VM、GBA ROM VM、多线程/channel VM | opcode stream、handler、状态寄存器、输入影响路径和可验证解释器 | [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md) |
| image XOR、bitmap convergence、Windows PE bitmap/OCR | 数据维度、颜色/通道、平滑性、OCR 字符集和 forward check | [file-signatures-and-flag-artifact-hunting.md](file-signatures-and-flag-artifact-hunting.md)、[image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md) |
| kernel module maze、custom binfmt、eBPF/tracepoint | 用户态入口和内核侧解释器/模块之间的数据结构 | [windows-kernel-ioctl-hidden-feedback-maze.md](windows-kernel-ioctl-hidden-feedback-maze.md)、[mobile-firmware-kernel-and-game-re.md](mobile-firmware-kernel-and-game-re.md) |
| shared library backdoor、hash-resolved imports、section header corruption | 加载器实际加载了哪个对象，工具失败是否来自 ELF/PE 元数据损坏 | [anti-analysis.md](anti-analysis.md)、[disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md) |
| game theory / byte-at-a-time / side-channel 状态 | 能否把执行反馈转为 oracle，而不是完整逆向全程序 | [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md) |

## 合并与拆分结论

- 保留为 family：raw 覆盖多个二阶段/特殊载体分支，核心价值是判断真实逻辑在哪里加载或被解释。
- 不合并进 VM family：VM family 是更高层判断，本页承接 loader、image、kernel module、binfmt 和 backdoor shared library 等具体载体。
- 不拆 image/bitmap 小页：当前 image 恢复案例和 reverse 载体关系更紧，先保留在本页并链接到 forensics 图片页。

## 常见误判

- 只分析第一阶段入口，忽略运行时解密、映射、重启、binfmt 或共享库替换。
- 工具无法解析 ELF/PE 就认为文件坏了；应先看 program header、原始二进制和加载器路径。
- VM 题急着完整还原 ISA，忘记 trace 热路径和最终比较点通常更快。
- 图片/bitmap 恢复没有记录宽高、通道顺序和字符集，导致结果不可复算。

## 关联页面

- [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md)
- [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md)
- [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md)
- [packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md)
- [windows-kernel-ioctl-hidden-feedback-maze.md](windows-kernel-ioctl-hidden-feedback-maze.md)
- [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md)

## 原始资料

- [loader-vm-image-and-kernel-patterns.md](../raw/reverse/loader-vm-image-and-kernel-patterns.md)
