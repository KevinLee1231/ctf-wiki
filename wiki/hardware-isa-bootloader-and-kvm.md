---
type: family
tags: [reverse, family, hardware, isa, bootloader, kvm, firmware]
skills: [ctf-reverse, ctf-pwn]
raw:
  - ../raw/reverse/hardware-isa-bootloader-and-kvm.md
  - ../raw/pwn/WMCTF2025-aberration-wp.md
updated: 2026-06-12
---

# Hardware ISA, Bootloader and KVM

## 作用边界

本页是低频架构、固件、ISA、bootloader、KVM/guest、微控制器和硬件协处理器 reverse family。它关注的是运行环境和指令语义本身：端序、加载基址、MMIO、特权级、调试接口、协处理器、SMC/TrustZone、KVM ioctl 或 boot ROM。

如果证据只是物理采样波形，转 [signals-and-hardware.md](signals-and-hardware.md)。如果已经形成内存破坏 primitive 或 secure world exploit，需同时回到 Pwn family。

## 识别信号

- 架构不是普通 x86/ARM 用户态：SPARC、RISC-V、MIPS64 OCTEON、Xtensa、Game Boy/Z80、MCU、MBR、bootloader、coreboot、TrustZone/EL3。
- 题目要求还原 MMIO register、GPIO、LCD 控制器、串口/I2C、固件加载地址、ROM 映射、特权指令或 KVM exit。
- 常规反编译不可用或误导，需要先确定端序、基址、调用约定、内存映射和执行环境。
- raw 或 WP 中出现硬件侧加密、SIMD lane、custom extension、SMC handler、guest/host 边界。

## 最小证据

- 架构、端序、位宽、加载基址和内存映射。
- 输入输出接口：UART/I2C/GPIO/MMIO/KVM ioctl/SMC/firmware table。
- 可运行或可模拟路径：QEMU、bgb、GDB stub、firmware emulator、trace log 或最小解释器。
- 要恢复的是算法、协议帧、显示输出、密钥、特权态数据，还是 exploit primitive。

## 路由表

| 证据 | 先验证 | 下一跳 |
|---|---|---|
| MCU/固件/MMIO AES | 寄存器地址、外设状态和 key/data 写入顺序 | 写 MMIO 状态模型或最小 emulator |
| LCD/GPIO/显示控制器 | 引脚到命令/数据线映射，时序和字符表 | 转 [signals-and-hardware.md](signals-and-hardware.md) 补物理层 |
| RISC-V/custom ISA | custom opcode、privileged mode、CSR 和调试通道 | 先扩反汇编/模拟器，再还原校验 |
| TrustZone/EL3 SMC | BL31/固件、SMC 参数、共享内存和 secure world 状态 | 转 [cross-primitive-escape-and-hybrid-exploit-map.md](cross-primitive-escape-and-hybrid-exploit-map.md) |
| MBR/bootloader/coreboot | 加载地址、实模式/保护模式切换和磁盘/ROM 布局 | [font-shader-firmware-and-legacy-patterns.md](font-shader-firmware-and-legacy-patterns.md) |
| KVM guest analysis | ioctl 参数、guest memory、KVM exit reason 和 halt/block 边界 | 记录 guest/host 数据流再反汇编 guest |
| SIMD/协处理器加密 | lane 排列、shuffle/gather、协处理器状态和常量表 | 先还原数据布局，再转 crypto/算法页 |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [WMCTF2025-aberration-wp](../raw/pwn/WMCTF2025-aberration-wp.md) | ARM64 secure machine 中 flag 在 EL3 系统寄存器，利用链是 BL31 SMC handler 竞态、管理字段破坏、页表权限伪造和 EL3 shellcode。 |
| [ACTF2026-acpu-wp](../raw/pwn/ACTF2026-acpu-wp.md) | CPU 仿真器题可能是微架构语义差异而非内存破坏；非法 load 的 forwarding/cache side effect 是关键证据。 |
| [ACTF2026-amcu-wp](../raw/pwn/ACTF2026-amcu-wp.md) | MCU 固件题要把串口、I2C、SRAM 和执行跳板放在同一个硬件状态模型里看。 |
| [LilacCTF2026-justrom-wp](../raw/reverse/LilacCTF2026-justrom-wp.md) | ROM/ISA 题先确认架构端序、加载基址和 MMIO register，再复现加密/比较函数。 |

## 合并与拆分结论

- 保留为 family：raw 跨 ISA、固件、bootloader、KVM、MCU、TrustZone 和协处理器，首要价值是运行环境分流。
- 不合并进 `font-shader-firmware-and-legacy-patterns.md`：该页是低频载体总入口，本页负责硬件/ISA 执行语义。
- 不合并进 Pwn kernel 页：只有进入漏洞利用 primitive 后才 pivot 到 Pwn。

## 常见误判

- 端序、基址或加载模式错了，后续算法恢复全部失真。
- 把 MMIO read/write 当普通内存，忽略外设状态机。
- TrustZone/EL3 题只看普通世界程序，没有提取 BL31/secure monitor。
- SIMD/协处理器题直接看标量伪代码，漏掉 lane 重排和硬件状态。

## 关联页面

- [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md)
- [font-shader-firmware-and-legacy-patterns.md](font-shader-firmware-and-legacy-patterns.md)
- [signals-and-hardware.md](signals-and-hardware.md)
- [mobile-firmware-kernel-and-game-re.md](mobile-firmware-kernel-and-game-re.md)
- [cross-primitive-escape-and-hybrid-exploit-map.md](cross-primitive-escape-and-hybrid-exploit-map.md)
- [disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md)

## 原始资料

- [hardware-isa-bootloader-and-kvm.md](../raw/reverse/hardware-isa-bootloader-and-kvm.md)
- [WMCTF2025-aberration-wp](../raw/pwn/WMCTF2025-aberration-wp.md)
