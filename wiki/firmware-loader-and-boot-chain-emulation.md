---
type: technique
tags: [reverse, hardware-embedded, firmware, bootloader, uefi, emulation]
skills: [ctf-reverse, ctf-hardware-embedded]
raw:
  - ../raw/reverse/hardware-isa-bootloader-and-kvm.md
  - ../raw/reverse/python-bytecode-esolangs-and-uefi.md
  - ../raw/reverse/UMDCTF2022-pi-trojan-wp.md
updated: 2026-07-28
---

# Firmware Loader and Boot-Chain Emulation

## 适用场景

附件是 boot sector、UEFI、MCU/SoC 裸机镜像或多阶段固件；核心是确定架构、装载基址、入口和阶段转换，再恢复运行时解密代码或以最小平台模型执行认证路径。

## 识别信号

- 文件没有 ELF/PE 头，但含 vector table、reset entry、boot header 或已知固件 magic。
- 运行地址与文件偏移不一致，必须通过装载基址换算。
- 早期代码初始化异常级别、栈、UART/MMIO 后跳入下一阶段。
- 入口写代码段、刷新 instruction cache 或原地解密后续认证函数。

## 最小证据

- 确认架构、端序、位宽、装载基址、入口与镜像切片边界。
- 用启动头或已知初始化指令验证反汇编选择。
- 运行地址到文件偏移的换算能定位至少一个已知常量/代码块。
- 模拟/stub 只覆盖认证路径实际使用的外设与异常行为。

## 解法骨架

1. 从平台约定、vector/header 和工具链痕迹确定 ISA 与基址。
2. 映射 ROM/RAM，跟踪 reset/boot stage 到输入或认证入口。
3. 离线复现原地 XOR/解压/重定位，或在执行后 dump 已解密阶段。
4. 对 UART、timer、storage 等必要 MMIO 建最小 stub。
5. 从最终比较、目标摘要或解密 buffer 提取结果，并按地址映射回原镜像验证。

## 关键变体

- BIOS/boot sector/UEFI stage。
- AArch64/ARM MCU 裸机镜像与 vector table。
- Secure monitor/TrustZone stage 切换。
- 多阶段 loader 与运行时自解密认证代码。

## 常见陷阱

- 装载基址错误时仍继续分析“似乎合理”的反汇编。
- 把运行地址直接当文件 offset。
- 看到大量非法指令就换架构，忽略入口正在解密代码段。
- 普通固件逻辑恢复误归硬件攻击；只有物理接口/信号决定解法时才转硬件专项路线。

## 关联技巧

- [custom-isa-and-mmio-emulation.md](custom-isa-and-mmio-emulation.md)
- [staged-loader-and-runtime-image-recovery.md](staged-loader-and-runtime-image-recovery.md)
- [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md)
- [python-bytecode-esolangs-and-uefi.md](python-bytecode-esolangs-and-uefi.md)

## 原始资料

- [hardware-isa-bootloader-and-kvm.md](../raw/reverse/hardware-isa-bootloader-and-kvm.md)
- [python-bytecode-esolangs-and-uefi.md](../raw/reverse/python-bytecode-esolangs-and-uefi.md)
- [UMDCTF2022-pi-trojan-wp](../raw/reverse/UMDCTF2022-pi-trojan-wp.md)
