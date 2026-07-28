---
type: technique
tags: [reverse, hardware-embedded, custom-isa, mmio, emulation]
skills: [ctf-reverse, ctf-hardware-embedded]
raw:
  - ../raw/reverse/hardware-isa-bootloader-and-kvm.md
  - ../raw/reverse/UMDCTF2023-clutter-wp.md
updated: 2026-07-28
---

# Custom ISA and MMIO Emulation

## 适用场景

附件使用低频/自定义 ISA、KVM guest、协处理器或少量 MMIO 外设；需要先恢复 opcode、寄存器和内存语义，再用最小解释器与外设 stub 重建校验或输出。

## 识别信号

- 常规反汇编器无法识别，且字节流呈固定长度或明显 opcode/operand 分布。
- 程序包含 PC、虚拟寄存器、tag、栈或自定义内存访问状态。
- 频繁访问少数固定 MMIO/hypercall 地址，返回值影响后续分支。
- 原程序无显式输出，但 verbose trace 中持续写入可疑小整数。

## 最小证据

- 由已知样本确认指令宽度、端序、寄存器数、PC 更新和至少一条指令语义。
- MMIO/hypercall stub 的输入输出有 trace 或调用上下文支撑。
- 解释器能在已知程序上重现原 trace、最终寄存器或内存状态。

## 解法骨架

1. 按字节频率和控制流候选划分 opcode/operand，先恢复 load/store/branch。
2. 写带 trace 的最小解释器，逐条对齐原程序的 PC 与状态变化。
3. 对未知外设按访问地址和数据流建立最小 stub，不模拟无关硬件。
4. 从目标内存区、输出端口或比较状态提取结果。
5. 必要时把稳定 opcode 语义提升为 SMT 约束，而不是直接约束原始字节。

## 常见陷阱

- 在多种 ISA 下“看起来像代码”就停止验证。
- PC 自增、符号扩展、位宽 wrap 或端序错误导致 trace 逐渐漂移。
- 为完整模拟所有外设耗时，却没有确认哪些寄存器到达校验路径。
- 把已知 boot image 装载问题混入本页；启动链应转固件 loader 技巧。

## 关联技巧

- [firmware-loader-and-boot-chain-emulation.md](firmware-loader-and-boot-chain-emulation.md)
- [custom-vm-and-wasm-state-lifting.md](custom-vm-and-wasm-state-lifting.md)
- [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md)
- [trace-hook-and-state-snapshot-reconstruction.md](trace-hook-and-state-snapshot-reconstruction.md)

## 原始资料

- [hardware-isa-bootloader-and-kvm.md](../raw/reverse/hardware-isa-bootloader-and-kvm.md)
- [UMDCTF2023-clutter-wp](../raw/reverse/UMDCTF2023-clutter-wp.md)
