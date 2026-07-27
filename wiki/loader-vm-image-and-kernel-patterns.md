---
type: family
tags: [reverse, family, loader, vm, image-recovery, kernel-module]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/loader-vm-image-and-kernel-patterns.md
  - ../raw/reverse/ACTF2026-abyssgate-wp.md
  - ../raw/reverse/西湖论剑2023-Berkeley-wp.md
updated: 2026-07-27
---

# Loader, VM, Image and Kernel Patterns

## 作用边界

本页是二阶段载体和特殊执行链 family，覆盖 loader、隐藏 opcode、binfmt、shared library backdoor、kernel module maze、image/bitmap 恢复、shellcode 数据段、GBA ROM VM、线程/通道 VM 和无导入/哈希解析样本。它不再作为 technique；它负责把“程序真实逻辑不在第一眼看到的入口”这类证据分流出去。

## 识别信号

- 第一阶段程序主要做解密、映射、释放、重启、注册 binfmt、加载共享库或挂载 probe，而不是直接校验 flag。
- 静态工具看到的入口、导入表、section header、opcode 或资源和运行时行为明显不一致。
- 数据段、图片、ROM、共享库、内核模块或 eBPF 程序在运行时才成为真实代码或解释器输入。
- 崩溃、trace、dump 或系统调用显示另一个执行层正在解释用户输入。

## 最小证据

- 标出真实逻辑出现的时机：解密后、`mmap` 后、`dlopen` 后、binfmt 触发后、kernel probe 触发后或 VM dispatch 中。
- 保存第一阶段到第二阶段之间的载荷位置、映射权限、入口地址、参数和校验数据流。
- 对 VM/解释器题，至少确认 opcode stream、handler 位置、状态变量和一个可回放输入到输出的路径。
- 对 image/bitmap/ROM 题，保存宽高、通道、地址映射、字符集或 forward checker，保证恢复结果可复算。

## 首轮路由

| 信号 | 先确认 | 下一跳 |
|---|---|---|
| 二阶段 loader、`mmap RWX`、数据段 shellcode、execve 递归 | 第一阶段做了什么解密/映射/重启，真实代码何时出现 | [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md)、[packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md) |
| 自定义 VM、GBA ROM VM、多线程/channel VM | opcode stream、handler、状态寄存器、输入影响路径和可验证解释器 | [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md) |
| image XOR、bitmap convergence、Windows PE bitmap/OCR | 数据维度、颜色/通道、平滑性、OCR 字符集和 forward check | [file-signatures-and-flag-artifact-hunting.md](file-signatures-and-flag-artifact-hunting.md)、[image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md) |
| kernel module maze、custom binfmt、eBPF/tracepoint | 用户态入口和内核侧解释器/模块之间的数据结构 | [windows-kernel-ioctl-hidden-feedback-maze.md](windows-kernel-ioctl-hidden-feedback-maze.md)、[mobile-firmware-kernel-and-game-re.md](mobile-firmware-kernel-and-game-re.md) |
| shared library backdoor、hash-resolved imports、section header corruption | 加载器实际加载了哪个对象，工具失败是否来自 ELF/PE 元数据损坏 | [anti-analysis.md](anti-analysis.md)、[disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md) |
| game theory / byte-at-a-time / side-channel 状态 | 能否把执行反馈转为 oracle，而不是完整逆向全程序 | [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md) |

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| Loader 解密/映射第二阶段，需在运行时转移前后 dump | [staged-loader-and-runtime-image-recovery.md](staged-loader-and-runtime-image-recovery.md) |
| boot/UEFI/kernel/低频 ISA 需恢复装载地址与模拟环境 | [unknown-isa-bootloader-and-firmware-emulation.md](unknown-isa-bootloader-and-firmware-emulation.md) |
| Loader 内嵌 Python/PYD 并由自定义校验器处理输入 | [embedded-python-pyd-custom-aes.md](embedded-python-pyd-custom-aes.md) |

## 合并与拆分结论

- 保留为 family：raw 覆盖多个二阶段/特殊载体分支，核心价值是判断真实逻辑在哪里加载或被解释。
- 不合并进 VM family：VM family 是更高层判断，本页承接 loader、image、kernel module、binfmt 和 backdoor shared library 等具体载体。
- 不拆 image/bitmap 小页：当前 image 恢复案例和 reverse 载体关系更紧，先保留在本页并链接到 forensics 图片页。

## 常见误判

- 只分析第一阶段入口，忽略运行时解密、映射、重启、binfmt 或共享库替换。
- 工具无法解析 ELF/PE 就认为文件坏了；应先看 program header、原始二进制和加载器路径。
- VM 题急着完整还原 ISA，忘记 trace 热路径和最终比较点通常更快。
- 图片/bitmap 恢复没有记录宽高、通道顺序和字符集，导致结果不可复算。

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [ACTF2026-abyssgate-wp](../raw/reverse/ACTF2026-abyssgate-wp.md) | 用户态 loader 释放第二阶段 ELF，eBPF 在 ioctl 进入/返回时改写参数，内核模块再完成协议状态机；必须把三层数据流合并建模。 |
| [西湖论剑2023-Berkeley-wp](../raw/reverse/西湖论剑2023-Berkeley-wp.md) | 用户态校验是诱饵，真实逻辑在 libbpf-bootstrap 挂载到自身 `check_flag` 的 uprobe/uretprobe 中；先提取 BPF 程序和 map 常量，再还原双层变换。 |
| [D3CTF2019-ch1pfs-wp](../raw/reverse/D3CTF2019-ch1pfs-wp.md) | 自定义文件系统镜像和 RC4 文件层加密，先恢复元数据结构和已知明文 keystream。 |
| [SUCTF2026-LockWP](../raw/reverse/SUCTF2026-LockWP.md) | Inno Setup、Rust overlay、锁屏程序和内核驱动多层嵌套；最终 IOCTL 中 XXTEA-like dword 校验在驱动层。 |
| [SUCTF2026-old_binWP](../raw/reverse/SUCTF2026-old_binWP.md) | 固件先 XOR 解包出 IMG0 容器，再修复损坏 ELF 和 TLS 布局；最后还原网络 challenge 与自定义块校验。 |
| [SUCTF2026-RevirdWP](../raw/reverse/SUCTF2026-RevirdWP.md) | 外层魔改 AES 解出第二阶段 EXE，随后通过 `\\.\Revird` 与驱动 op case 协同完成 AES-like 校验。 |
| [VNCTF2026-shadow-wp](../raw/reverse/VNCTF2026-shadow-wp.md) | 用户态迷宫只触发 `Sleep(0x32)`；真实校验在反射加载驱动、PTE hook、键盘记录和基于 `KeDelayExecutionThread` 参数解密的 shellcode。 |
| [0xGame2025-week3-easyApp-wp](../raw/reverse/0xGame2025-week3-easyApp-wp.md) | 本题的校验链为“长度 42 → 前 26 字符 Base64 → 后 16 字符转十六进制 → 动态 Dex 反射调用 → 三元一次方程组”。主 Dex 只负责调度，真正的第二阶段约束位于 assets 中，因此看到 `getAssets().open`、`DexClassLoader`、`loadClass` 和 `getDeclaredMethod` 时，应继续提取被加载的文件，不能停在主程序。 |
| [MoeCTF2021-pepepe-wp](../raw/reverse/MoeCTF2021-pepepe-wp.md) | 修复 DOS 头的 `e_lfanew`，再利用 PE DOS stub 的稳定文本对重复异或密钥做已知明文恢复；文件以 `MZ` 开头但工具找不到 `PE\0\0` 时，应优先检查 `0x3c` 指针；密文目标仍是 PE 时，可尝试标准 stub 字符串和固定文件头。 |
| [MoeCTF2023-unwind-wp](../raw/reverse/MoeCTF2023-unwind-wp.md) | x86 SEH handler 不只可能在“寻找处理器”阶段执行，也可能在栈展开阶段再次执行。逆向异常驱动程序时，应同时跟踪 handler 返回值、外层处理器选择和 unwind 路径；只看反编译器展示的普通调用图，会漏掉本题后半段的第二次 TEA。 |
| [SekaiCTF2026-Chibile-wp](../raw/reverse/SekaiCTF2026-Chibile-wp.md) | 本题的决定性障碍是多模块共同实现的私有协议，而不是驱动利用。先把 gate、attestation、秘密 KDF 和外层传输拆成四层，标清每层的密钥、nonce 与字节序，便可逐层验证。遇到壳内常量时，不必只依赖静态搜索；“内存中的 32 字节窗口能否把真实会话密文解成受限字符集”本身就是很强的密钥判定 oracle。 |
| [UMDCTF2020-ez-pe-wp](../raw/reverse/UMDCTF2020-ez-pe-wp.md) | 修复损坏 PE 时应从结构层级逐步验证：先签名，再 COFF 头和可选头，最后才看节表与导入。机器类型与可选头位数必须一致，否则即使签名正确，系统加载器仍会拒绝文件。 |
| [UMDCTF2022-pi-trojan-wp](../raw/reverse/UMDCTF2022-pi-trojan-wp.md) | 裸机镜像应先确定装载基址，再把运行地址换算成文件偏移。遇到大量看似非法指令时，不应立即认定架构错误；本题入口明确执行了写代码段和刷新指令缓存，说明后续代码是运行时解密的。两层 XOR 还原后，最终校验只是普通 SHA-256 比较，提取目标摘要即可，不需要尝试不可行的通用哈希逆向。 |
| [WMCTF2020-ipcam-wp](../raw/reverse/WMCTF2020-ipcam-wp.md) | 固件题不要只依赖 `binwalk -e`，自动解包失败时应先检查分区块的字节序、偏移和文件头。遇到大端序块时，先 `dd` 提取再用 `objcopy` 转端序，随后逆向关键服务程序。IoT 题的 flag 也可能分布在多个服务面：本题第一段来自 `p2pcam` 中的漏洞利用，第二段则需要注意扫描到的 554/RTSP 服务。 |
| [WMCTF2020-wmware-wp](../raw/reverse/WMCTF2020-wmware-wp.md) | 从 Bochs 配置定位虚拟磁盘，按 MBR 扇区加载关系拆出 `0x400/0xc00/0x1400` 三段，区分 16 位和 32 位反汇编，逆向键盘通码、换位、加 `0x55` 和 129 轮异或校验。 |
| [WMCTF2022-babydriver-wp](../raw/reverse/WMCTF2022-babydriver-wp.md) | 先从用户态程序定位输入长度、预处理和驱动释放行为，再在驱动中分析回调通信与 AES 校验逻辑；Reverse 附件名、题目名和程序行为都指向 driver，且用户态参数最终进入 `NtQueryInformationFile` 这类系统调用时，应检查是否借助内核回调或 hook 建立 R3/R0 通道。 |

## 关联页面

- [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md)
- [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md)
- [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md)
- [packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md)
- [windows-kernel-ioctl-hidden-feedback-maze.md](windows-kernel-ioctl-hidden-feedback-maze.md)
- [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md)

## 原始资料

- [loader-vm-image-and-kernel-patterns.md](../raw/reverse/loader-vm-image-and-kernel-patterns.md)
- [ACTF2026-abyssgate-wp](../raw/reverse/ACTF2026-abyssgate-wp.md)
- [西湖论剑2023-Berkeley-wp](../raw/reverse/西湖论剑2023-Berkeley-wp.md)
