---
type: family
tags: [reverse, family, font, shader, legacy-format, side-channel]
skills: [ctf-reverse, ctf-misc]
raw:
  - ../raw/reverse/font-shader-firmware-and-legacy-patterns.md
updated: 2026-06-12
---

# Font, Shader, Firmware and Legacy Patterns

## 作用边界

本页是特殊载体与低频模式 family，覆盖字体 ligature、GLSL shader VM、LED/ioctl 摩斯、BPF JIT、ESP32/Xtensa、MBR、BWT、滑窗 popcount、单行 Python 约束、batch crackme、time lock、fork/pipe 反分析等案例。共同点是：主要障碍不是普通反编译，而是先识别隐藏信息承载方式或可观测副作用。

## 首轮路由

| 信号 | 先确认 | 下一跳 |
|---|---|---|
| OpenType/字体/ligature/私用区字符 | 字体表、glyph substitution、动态加载字体和文本渲染路径 | [misc-cross-category-triage-family.md](misc-cross-category-triage-family.md) |
| GLSL/shader/VRAM/图形 VM | shader 状态、self-modifying store、输出图像和可 trace 的渲染路径 | [loader-vm-image-and-kernel-patterns.md](loader-vm-image-and-kernel-patterns.md) |
| LED/ioctl、BPF JIT、syscall side effect | 系统调用、设备状态、kernel log 或 side effect 是否承载输出 | [qiling-triton-pin-and-ldpreload.md](qiling-triton-pin-and-ldpreload.md)、[mobile-firmware-kernel-and-game-re.md](mobile-firmware-kernel-and-game-re.md) |
| BWT、popcount、single-line Python/Z3、DNN inversion | 能否把变换转为约束、逆变换或小范围枚举 | [self-decrypting-strings-and-lattice-patterns.md](self-decrypting-strings-and-lattice-patterns.md) |
| MBR/16-bit/legacy executable/ESP32/Xtensa | 架构、加载基址、固件符号表和执行环境是否先于算法 | [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md) |
| fork/pipe/dead branch/time lock/batch crackme | 是否可通过 patch、faketime、pattern extraction 或自动化批处理降维 | [anti-analysis.md](anti-analysis.md)、[runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md) |

## 合并与拆分结论

- 保留为 family：raw 覆盖的格式和副作用差异很大，但共同价值是提示“别把它当普通二进制读”。
- 不合并进 Misc family：本页仍有大量 reverse 执行模型和工具链判断；Misc family 只作为跨方向入口。
- 暂不拆字体/shader/BPF 子页：当前案例数量不足以支撑独立 technique，先通过分流表链接到更具体页面。

## 常见误判

- 字体题只看文本文件，不检查字体替换、ligature 和渲染时行为。
- shader/VRAM 题只看输出截图，没有 trace store、坐标和颜色状态。
- side-channel 题没有记录采样规则和噪声边界，导致结果不可复现。
- 低频架构题没先确认端序、基址和调用约定，直接读错反汇编。

## 关联页面

- [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md)
- [loader-vm-image-and-kernel-patterns.md](loader-vm-image-and-kernel-patterns.md)
- [self-decrypting-strings-and-lattice-patterns.md](self-decrypting-strings-and-lattice-patterns.md)
- [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md)
- [anti-analysis.md](anti-analysis.md)
- [misc-cross-category-triage-family.md](misc-cross-category-triage-family.md)

## 原始资料

- [font-shader-firmware-and-legacy-patterns.md](../raw/reverse/font-shader-firmware-and-legacy-patterns.md)
