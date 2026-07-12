---
type: family
tags: [reverse, family, font, shader, legacy-format, side-channel]
skills: [ctf-reverse, ctf-hardware-embedded, ctf-stego]
raw:
  - ../raw/reverse/font-shader-firmware-and-legacy-patterns.md
updated: 2026-07-06
---

# Font, Shader, Firmware and Legacy Patterns

## 作用边界

本页是特殊载体与低频模式 family，覆盖字体 ligature、GLSL shader VM、LED/ioctl 摩斯、BPF JIT、ESP32/Xtensa、MBR、BWT、滑窗 popcount、单行 Python 约束、batch crackme、time lock、fork/pipe 反分析等案例。共同点是：主要障碍不是普通反编译，而是先识别隐藏信息承载方式或可观测副作用。

## 识别信号

- 附件或题面出现字体、shader、VRAM、LED/ioctl、BPF/JIT、MBR/16-bit、ESP32/Xtensa、BWT/popcount、batch/time lock 等低频载体。
- 可复用信息不一定在普通函数伪代码里，而可能在渲染替换、图形状态、设备 side effect、采样序列、固件加载基址或脚本约束中。
- 题目需要先识别“信息如何被承载或观察”，再决定是否转 VM、Hardware/Embedded、反分析、Stego 或其它正式专项。
- raw 证据通常包含截图、glyph 表、shader 输出、系统调用、日志、设备状态、采样轨迹或非标准格式字段。

## 最小证据

- 保存载体类型、打开/渲染/运行方式、可观察输出和一个能复现隐藏信息或副作用的最小样本。
- 字体题至少确认字体表、glyph substitution/ligature、动态加载路径和文本渲染前后的差异。
- shader/VRAM 题至少记录坐标、颜色/纹理状态、store/load 语义和输出图像如何对应内部状态。
- side-channel/legacy/固件题至少确认架构、端序、加载基址、采样规则、噪声边界或设备接口。

## 首轮路由

| 信号 | 先确认 | 下一跳 |
|---|---|---|
| OpenType/字体/ligature/私用区字符 | 字体表、glyph substitution、动态加载字体和文本渲染路径 | [cross-category-triage-family.md](cross-category-triage-family.md) |
| GLSL/shader/VRAM/图形 VM | shader 状态、self-modifying store、输出图像和可 trace 的渲染路径 | [loader-vm-image-and-kernel-patterns.md](loader-vm-image-and-kernel-patterns.md) |
| LED/ioctl、BPF JIT、syscall side effect | 系统调用、设备状态、kernel log 或 side effect 是否承载输出 | [qiling-triton-pin-and-ldpreload.md](qiling-triton-pin-and-ldpreload.md)、[mobile-firmware-kernel-and-game-re.md](mobile-firmware-kernel-and-game-re.md) |
| BWT、popcount、single-line Python/Z3、DNN inversion | 能否把变换转为约束、逆变换或小范围枚举 | [self-decrypting-strings-and-lattice-patterns.md](self-decrypting-strings-and-lattice-patterns.md) |
| MBR/16-bit/legacy executable/ESP32/Xtensa | 架构、加载基址、固件符号表和执行环境是否先于算法 | [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md) |
| fork/pipe/dead branch/time lock/batch crackme | 是否可通过 patch、faketime、pattern extraction 或自动化批处理降维 | [anti-analysis.md](anti-analysis.md)、[runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md) |

## 合并与拆分结论

- 保留为 family：raw 覆盖的格式和副作用差异很大，但共同价值是提示“别把它当普通二进制读”。
- 不合并进跨方向总入口：本页仍有大量 Reverse 执行模型和工具链判断；总入口只负责首轮分流。
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
- [cross-category-triage-family.md](cross-category-triage-family.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [D3CTF2019-c-c-wp](../raw/reverse/D3CTF2019-c-c-wp.md) | 私用区字符依赖动态加载字体，先脱壳跟进字体 hook 并 dump 解密后的字体资源。 |

## 原始资料

- [font-shader-firmware-and-legacy-patterns.md](../raw/reverse/font-shader-firmware-and-legacy-patterns.md)
