---
type: technique
tags: [reverse, hardware-embedded, isa, bootloader, firmware, emulation]
skills: [ctf-reverse, ctf-hardware-embedded]
raw:
  - ../raw/reverse/hardware-isa-bootloader-and-kvm.md
  - ../raw/reverse/python-bytecode-esolangs-and-uefi.md
updated: 2026-07-27
---

# Unknown ISA, Bootloader and Firmware Emulation

## 适用场景

附件是低频/自定义 ISA、boot sector、UEFI、MCU 固件或 KVM guest code；需要先确定装载地址、指令语义和外设映射，再以 emulator/harness 重建行为。

## 识别信号

- 常规反汇编器无法识别，或同一字节在多个 ISA 下均似合理。
- 固件有 vector table、reset entry、boot header、MMIO 地址或阶段加载。
- 程序依赖少量外设寄存器、hypercall 或自定义协处理器。

## 最小证据

- 确认架构、端序、位宽、加载基址和入口。
- 用已知指令/启动头验证解码。
- Stub 的 MMIO/hypercall 输入输出有 trace 支撑。

## 解法骨架

1. 从 header、vector、工具链字符串和 opcode 分布识别 ISA。
2. 映射 ROM/RAM/MMIO，先执行到稳定循环或输入点。
3. 对未知外设按访问 trace 建最小行为 stub。
4. 导出关键状态转移，必要时提升到约束求解。

## 关键变体

- Boot sector/UEFI stage。
- MCU/TrustZone/secure monitor。
- Custom ISA/KVM device/co-processor。

## 常见陷阱

- 基址或 Thumb/ARM/端序选错，反汇编仍“像代码”。
- 为完整模拟所有外设浪费时间，未聚焦校验路径。
- 把普通固件逆向误归硬件方向，未检查物理接口是否决定解法。

## 关联技巧

- [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md)
- [python-bytecode-esolangs-and-uefi.md](python-bytecode-esolangs-and-uefi.md)
- [custom-vm-and-wasm-state-lifting.md](custom-vm-and-wasm-state-lifting.md)

## 原始资料

- [hardware-isa-bootloader-and-kvm.md](../raw/reverse/hardware-isa-bootloader-and-kvm.md)
- [python-bytecode-esolangs-and-uefi.md](../raw/reverse/python-bytecode-esolangs-and-uefi.md)
