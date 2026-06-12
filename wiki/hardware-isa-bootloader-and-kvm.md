---
type: technique
tags: [reverse, technique]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/hardware-isa-bootloader-and-kvm.md
updated: 2026-05-21
---

# Hardware ISA, Bootloader and KVM

## 适用场景

需要理解二进制、脚本、字节码、壳、VM、固件或混淆逻辑，再恢复算法或输入。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 附件是 ELF/PE/APK/WASM/pyc/固件/脚本，或存在壳、SMC、自定义 VM。
- flag 校验藏在运行时生成代码、解密字符串、解释器或 native 扩展中。
- 静态字符串不足，需要交叉引用、动态断点或 trace。
- 题面或 raw 线索能落到这些关键词之一：HD44780 LCD Controller GPIO Reconstruction (32C3 2015)、RISC-V (Advanced)、Custom Extensions、Privileged Modes、RISC-V Debugging、ARM64/AArch64 Reversing and Exploitation、MIPS64 Cavium OCTEON Coprocessor 2 Crypto (SEC-T CTF 2017)、EFM32 ARM Microcontroller MMIO AES (SEC-T CTF 2017)。

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
| HD44780 LCD Controller GPIO Reconstruction (32C3 2015) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| RISC-V (Advanced) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Custom Extensions | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Privileged Modes | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| RISC-V Debugging | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| ARM64/AArch64 Reversing and Exploitation | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| MIPS64 Cavium OCTEON Coprocessor 2 Crypto (SEC-T CTF 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| EFM32 ARM Microcontroller MMIO AES (SEC-T CTF 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| MBR/Bootloader Reversing with QEMU + GDB (Square CTF 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Game Boy ROM Z80 Analysis in bgb Debugger (Square CTF 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| KVM Guest Analysis via ioctl + KVMEXITHLT Block Chaining (CSAW 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Coreboot ROM XOR-Pair Bit-Flip Address Discovery (Hack.lu 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

## 常见陷阱

- 只按关键词跳页，没有先构造最小证据。
- 照搬 raw 中的一次性 payload，没有检查当前题的边界条件。
- 忽略相邻技巧之间的 pivot，导致在错误方向上继续投入时间。

## 关联技巧

- [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)
- [anti-analysis.md](anti-analysis.md)
- [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md)
- [disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md)
- [embedded-python-pyd-custom-aes.md](embedded-python-pyd-custom-aes.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [ACTF2026-acpu-wp](../raw/pwn/ACTF2026-acpu-wp.md) | CPU 仿真器题可能是微架构语义差异而非内存破坏；非法 load 的 forwarding/cache side effect 是关键证据。 |
| [ACTF2026-amcu-wp](../raw/pwn/ACTF2026-amcu-wp.md) | MCU 固件题要把串口、I2C、SRAM 和执行跳板放在同一个硬件状态模型里看。 |
| [LilacCTF2026-justrom-wp](../raw/reverse/LilacCTF2026-justrom-wp.md) | ROM/ISA 题先确认架构端序、加载基址和 MMIO register，再复现加密/比较函数。 |
| [D3CTF2019-easy-dongle-wp](../raw/reverse/D3CTF2019-easy-dongle-wp.md) | 加密狗/STM32 固件通过 UART 与主程序协作时，协议帧和固件加载地址同等关键。 |
| [D3CTF2019-simd-wp](../raw/reverse/D3CTF2019-simd-wp.md) | SIMD 并行校验题先恢复 lane 排列、shuffle/gather 语义，再考虑密码算法本身。 |

## 原始资料

- [hardware-isa-bootloader-and-kvm.md](../raw/reverse/hardware-isa-bootloader-and-kvm.md)
