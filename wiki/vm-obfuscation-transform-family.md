---
type: family
tags: [reverse, family, vm, obfuscation, bytecode, smc, anti-debug, tracing]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/vm-obfuscation-transform-patterns.md
updated: 2026-07-06
---

# VM、混淆与字节变换技巧族

## 作用边界

本页是 Reverse 中“执行模型被隐藏或改写”的 family，负责把自定义 VM、flattening、runtime 解密、SMC、反调试和字节变换分流到合适的 trace、lifting、patch、oracle 或约束恢复路线。

它不要求完整反混淆；首要目标是找到真实校验边界、可观测中间状态和 forward check。如果障碍只是普通文件格式、语言运行时或工具使用，应先回到对应 family / tooling。

## 识别信号

- 程序存在自定义 bytecode、dispatch loop、状态机、加壳、自修改代码、runtime 解密、反调试或异常控制流。
- 静态反编译结果充满无意义分支、巨大 switch、间接跳转、flattening、nanomites 或大量相似 handler。
- 输入校验最终表现为字节变换、约束系统、解释器执行、trace 比较或运行时生成代码。
- 附件可能是 PE/ELF/APK/WASM/pyc/固件/游戏客户端；障碍是“先理解执行模型”。

## 最小证据

- 已定位真实入口、校验边界、比较点、dispatch loop、handler 表或 runtime 解密点之一。
- 能采集最小 trace：输入字节如何影响状态、寄存器、内存、handler 序列或最终比较 buffer。
- 已判断适合路线：静态 lifting、动态 trace、hook/patch、fuzz ISA、符号执行、直接抓明文或约束求解。
- 有 forward check：恢复出的输入能在原程序、patch 后程序或等价解释器中通过。

## 分流流程

1. 首检载体、架构、入口、导入、字符串、反调试和运行需求。
2. 找真实校验边界：最终 compare、成功/失败分支、解密后明文、VM dispatch、handler 表。
3. 选择降维路线：hook 输出、patch 反调试、trace handler、lift bytecode、提取变换、Z3/angr 求解。
4. 优先做最小可验证模型，不追求完整还原全程序。
5. 用原程序或自写 forward checker 验证 flag；记录关键断点、地址、脚本和失败分支。

## 路线分流

| 变体 | 优先证据 | 下一跳页面 | 失败后 pivot |
|---|---|---|---|
| 自定义 VM / bytecode | dispatch loop、handler 表、opcode stream | [loader-vm-image-and-kernel-patterns.md](loader-vm-image-and-kernel-patterns.md) | handler 太多时先 trace 热路径或 fuzz ISA，不完整反编译。 |
| Flattening / 状态机 | 单大循环、state variable、间接跳转 | [qiling-triton-pin-and-ldpreload.md](qiling-triton-pin-and-ldpreload.md) | 静态难读时转 trace、patch dispatcher 或识别状态转移表。 |
| 自修改 / runtime 解密 | 写代码段、解密字符串、JIT buffer | [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md) | 静态字符串无效时断在解密后使用点。 |
| 反调试 / nanomites | ptrace、IsDebuggerPresent、异常回调、debug event | [anti-analysis.md](anti-analysis.md) | 绕不过时改为模拟、patch 条件、record/replay 或外部 oracle。 |
| 字节变换 / 约束求解 | XOR/add/rotate/sbox/lattice/多轮比较 | [self-decrypting-strings-and-lattice-patterns.md](self-decrypting-strings-and-lattice-patterns.md) | 约束太大时先切分 block、抓中间明文或用 meet-in-the-middle。 |
| 混合模式 / kernel / loader | WoW64 far jump、驱动 IOCTL、binfmt、loader | [windows-kernel-ioctl-hidden-feedback-maze.md](windows-kernel-ioctl-hidden-feedback-maze.md) | 如果目标转为内存破坏或提权，再 pivot 到 pwn/kernel 页面。 |

## 常见陷阱

- 追求完整反混淆，忘记 CTF 目标通常只需要恢复输入约束或明文。
- 自动反编译失败后继续读伪代码；应切到汇编、trace、hook 或 emulator。
- 只看失败分支，不断在成功/比较点抓最终 buffer。
- patch 反调试后没有确认是否改变校验语义。
- 解出结果后不做 forward check，导致把中间态或编码态误当 flag。


## 关联技巧

- [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)
- [anti-analysis.md](anti-analysis.md)
- [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md)
- [disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md)
- [embedded-python-pyd-custom-aes.md](embedded-python-pyd-custom-aes.md)
- [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md)
- [go-rust-jvm-and-cpp-reversing.md](go-rust-jvm-and-cpp-reversing.md)
- [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md)
- [mobile-firmware-kernel-and-game-re.md](mobile-firmware-kernel-and-game-re.md)
- [signal-trace-and-packed-anti-analysis.md](signal-trace-and-packed-anti-analysis.md)
- [windows-kernel-ioctl-hidden-feedback-maze.md](windows-kernel-ioctl-hidden-feedback-maze.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [ACTF2026-计算机系统贯通实验-wp](../raw/reverse/ACTF2026-计算机系统贯通实验-wp.md) | xlsx 公式实现 RISC-V 单周期 CPU；先解包 XML 恢复 ROM/RAM/MMIO、指令解码和输出语义，再拆四段校验。 |
| [ACTF2026-virtualnpu-wp](../raw/reverse/ACTF2026-virtualnpu-wp.md) | CUDA fatbin 中宿主先解出 NPU bytecode；提取 `MOV_IMM` 比较常量后逆 RC4 drop-512 和多层 S-box/XOR 变换。 |
| [Bugku-week4_re1-wp](../raw/reverse/Bugku-week4_re1-wp.md) | 简单 VM 执行固定长度字节码，输入/寄存器/输出数组模式重复；可直接抽象每字符通项公式再正向验证。 |
| [D3CTF2023-d3sky-wp](../raw/reverse/D3CTF2023-d3sky-wp.md) | TLS 反调试和 RC4 加密 opcode 的自修改 VM，先恢复正确异常路径和滑动 XOR 关系。 |
| [D3CTF2023-d3syscall-wp](../raw/reverse/D3CTF2023-d3syscall-wp.md) | 内核模块把保留 syscall 映射成 VM 指令，先 strace 参数并恢复 syscall 到 opcode 的语义。 |
| [HGAME2026-androuge-wp](../raw/reverse/HGAME2026-androuge-wp.md) | APK 释放魔改 Lua 5.4 VM 与加密 `game` bytecode；先还原 XOR 载入层和 opcode 位域，再提密文数组与 seed。 |
| [NCTF2026-vm-encryptor-wp](../raw/reverse/NCTF2026-vm-encryptor-wp.md) | 先写自定义 VM disassembler 理清 opcode；真实算法是循环位移/XOR 后进魔改 Base64，再整体 XOR。 |
| [RCTF2025-onion-wp](../raw/reverse/RCTF2025-onion-wp.md) | 自定义 VM 有 PC/HIPC/LOTAG/HITAG/虚拟栈和 50 个 64-bit 输入；先实现反汇编/解释器，再把每个 check 自动逆算。 |
| [SUCTF2026-MvsicPlayerWP](../raw/reverse/SUCTF2026-MvsicPlayerWP.md) | Electron 音乐播放器先解析 `.su_mv` payload，再由 native `.node` 对 WAV 分支执行 VM bytecode 加密；目标是恢复原 WAV MD5。 |

## 原始资料

- [vm-obfuscation-transform-patterns.md](../raw/reverse/vm-obfuscation-transform-patterns.md)
