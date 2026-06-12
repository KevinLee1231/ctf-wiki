---
type: family
tags: [reverse, family, triage]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/reverse-first-pass-workflow-and-debugging.md
updated: 2026-06-11
---

# First-Pass Workflow and Debugging

## 适用场景

需要理解二进制、脚本、字节码、壳、VM、固件或混淆逻辑，再恢复算法或输入。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 附件是 ELF/PE/APK/WASM/pyc/固件/脚本，或存在壳、SMC、自定义 VM。
- flag 校验藏在运行时生成代码、解密字符串、解释器或 native 扩展中。
- 静态字符串不足，需要交叉引用、动态断点或 trace。
- 题面或 raw 线索能落到这些关键词之一：Problem-Solving Workflow、逆向首轮快判技巧族：静态、动态、dump 与误导识别、首轮低成本尝试、Initial Analysis、Memory Dumping Strategy、Decoy Flag Detection、GDB PIE Debugging、Comparison Direction (Critical!)。

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
| Problem-Solving Workflow | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| 逆向首轮快判技巧族：静态、动态、dump 与误导识别 | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| 首轮低成本尝试 | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Initial Analysis | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Memory Dumping Strategy | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Decoy Flag Detection | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| GDB PIE Debugging | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Comparison Direction (Critical!) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| 常用工具速查 | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| 深入分析转向条件 | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Expected Values Tables | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Unstripped Binary Information Leaks | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

## 常见陷阱

- 只按关键词跳页，没有先构造最小证据。
- 照搬 raw 中的一次性 payload，没有检查当前题的边界条件。
- 忽略相邻技巧之间的 pivot，导致在错误方向上继续投入时间。

## 关联技巧

- [windows-kernel-ioctl-hidden-feedback-maze.md](windows-kernel-ioctl-hidden-feedback-maze.md)
- [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md)
- [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)
- [anti-analysis.md](anti-analysis.md)
- [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md)
- [disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md)
- [embedded-python-pyd-custom-aes.md](embedded-python-pyd-custom-aes.md)
- [go-rust-jvm-and-cpp-reversing.md](go-rust-jvm-and-cpp-reversing.md)
- [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md)
- [mobile-firmware-kernel-and-game-re.md](mobile-firmware-kernel-and-game-re.md)
- [signal-trace-and-packed-anti-analysis.md](signal-trace-and-packed-anti-analysis.md)
- [loader-vm-image-and-kernel-patterns.md](loader-vm-image-and-kernel-patterns.md)

- [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md)
- [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md)
- [rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md)
- [scripts-and-obfuscation.md](scripts-and-obfuscation.md)
- [vm-z3-sandbox-and-game-basics.md](vm-z3-sandbox-and-game-basics.md)

## WP 案例沉淀

本节只抽取 raw WP 中可复用的识别信号和下一跳，不替代原始题解正文。

| Raw WP | 复用信号 | 下一跳 |
|---|---|---|
| [ACTF2026-计算机系统贯通实验-wp](../raw/reverse/ACTF2026-计算机系统贯通实验-wp.md) | 常规逆向首轮，先做载体、字符串、交叉引用、动态断点和最小解密脚本。 | [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md) |
| [ACTF2026-abyssgate-wp](../raw/reverse/ACTF2026-abyssgate-wp.md) | 用户态 loader、二阶段 ELF、eBPF tracepoint 和内核模块共同校验，先合并 ioctl 参数改写和内核数据流。 | [loader-vm-image-and-kernel-patterns.md](loader-vm-image-and-kernel-patterns.md) |
| [ACTF2026-calc-my-point-wp](../raw/reverse/ACTF2026-calc-my-point-wp.md) | Rust/GMP/大整数表达式执行器是主边界，先恢复语言运行时对象、数值约束和 CRT 关系。 | [go-rust-jvm-and-cpp-reversing.md](go-rust-jvm-and-cpp-reversing.md) |
| [ACTF2026-flagchecker-wp](../raw/reverse/ACTF2026-flagchecker-wp.md) | 复杂校验最终落到比较点，先断在比较/输出前恢复明文或中间 buffer。 | [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md) |
| [ACTF2026-virtualnpu-wp](../raw/reverse/ACTF2026-virtualnpu-wp.md) | VM/解释器/混淆变换是主障碍，先定位 dispatch、状态寄存器、opcode 和真实比较点。 | [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md) |
| [Bugku-BabyRE-wp](../raw/reverse/Bugku-BabyRE-wp.md) | 复杂校验最终落到比较点，先断在比较/输出前恢复明文或中间 buffer。 | [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md) |
| [Bugku-Berkeley-wp](../raw/reverse/Bugku-Berkeley-wp.md) | eBPF uprobe/uretprobe 把真实校验藏到内核侧 instrumentation，先提取 BPF 程序、hook 点和常量表。 | [qiling-triton-pin-and-ldpreload.md](qiling-triton-pin-and-ldpreload.md) |
| [Bugku-Dual-Personality-wp](../raw/reverse/Bugku-Dual-Personality-wp.md) | 常规逆向首轮，先做载体、字符串、交叉引用、动态断点和最小解密脚本。 | [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md) |
| [Bugku-EasyVT-wp](../raw/reverse/Bugku-EasyVT-wp.md) | VM/解释器/混淆变换是主障碍，先定位 dispatch、状态寄存器、opcode 和真实比较点。 | [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md) |
| [Bugku-foolme-wp](../raw/reverse/Bugku-foolme-wp.md) | 早期 sleep/exit 和 decoy target 误导动态运行，先区分反分析门槛与真正静态校验逻辑。 | [anti-analysis.md](anti-analysis.md) |
| [Bugku-GoldDigger-wp](../raw/reverse/Bugku-GoldDigger-wp.md) | 常规逆向首轮，先做载体、字符串、交叉引用、动态断点和最小解密脚本。 | [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md) |
| [Bugku-JustRe-wp](../raw/reverse/Bugku-JustRe-wp.md) | 复杂校验最终落到比较点，先断在比较/输出前恢复明文或中间 buffer。 | [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md) |
| [Bugku-MaybeNotStandrad-wp](../raw/reverse/Bugku-MaybeNotStandrad-wp.md) | 常规逆向首轮，先做载体、字符串、交叉引用、动态断点和最小解密脚本。 | [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md) |
| [Bugku-Roundabout-wp](../raw/reverse/Bugku-Roundabout-wp.md) | UPX/壳是首要边界，先脱壳或 dump 后再处理 XOR/key 与比较表。 | [packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md) |
| [Bugku-week1_re1-wp](../raw/reverse/Bugku-week1_re1-wp.md) | 复杂校验最终落到比较点，先断在比较/输出前恢复明文或中间 buffer。 | [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md) |
| [Bugku-week1_re2-wp](../raw/reverse/Bugku-week1_re2-wp.md) | 复杂校验最终落到比较点，先断在比较/输出前恢复明文或中间 buffer。 | [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md) |
| [Bugku-week1_re3-wp](../raw/reverse/Bugku-week1_re3-wp.md) | 复杂校验最终落到比较点，先断在比较/输出前恢复明文或中间 buffer。 | [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md) |
| [Bugku-week1_re4-wp](../raw/reverse/Bugku-week1_re4-wp.md) | 复杂校验最终落到比较点，先断在比较/输出前恢复明文或中间 buffer。 | [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md) |
| [Bugku-week2_re1-wp](../raw/reverse/Bugku-week2_re1-wp.md) | 只给汇编文本和数组时，先把指令语义转成可复算脚本，而不是寻找运行时断点。 | [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md) |
| [Bugku-week2_re2-wp](../raw/reverse/Bugku-week2_re2-wp.md) | 复杂校验最终落到比较点，先断在比较/输出前恢复明文或中间 buffer。 | [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md) |
| [Bugku-week2_re3-wp](../raw/reverse/Bugku-week2_re3-wp.md) | 复杂校验最终落到比较点，先断在比较/输出前恢复明文或中间 buffer。 | [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md) |
| [Bugku-week2_re4-wp](../raw/reverse/Bugku-week2_re4-wp.md) | PyInstaller/Python 编码逻辑是主边界，先恢复 Python 代码、常量和自定义 Base58 变换。 | [python-bytecode-esolangs-and-uefi.md](python-bytecode-esolangs-and-uefi.md) |
| [Bugku-week3_re1-wp](../raw/reverse/Bugku-week3_re1-wp.md) | 复杂校验最终落到比较点，先断在比较/输出前恢复明文或中间 buffer。 | [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md) |
| [Bugku-week3_re2-wp](../raw/reverse/Bugku-week3_re2-wp.md) | 复杂校验最终落到比较点，先断在比较/输出前恢复明文或中间 buffer。 | [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md) |
| [Bugku-week3_re3-wp](../raw/reverse/Bugku-week3_re3-wp.md) | 复杂校验最终落到比较点，先断在比较/输出前恢复明文或中间 buffer。 | [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md) |
| [Bugku-week4_re1-wp](../raw/reverse/Bugku-week4_re1-wp.md) | 复杂校验最终落到比较点，先断在比较/输出前恢复明文或中间 buffer。 | [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md) |
| [Bugku-week4_re2-wp](../raw/reverse/Bugku-week4_re2-wp.md) | 复杂校验最终落到比较点，先断在比较/输出前恢复明文或中间 buffer。 | [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md) |
| [Bugku-week4_re3-wp](../raw/reverse/Bugku-week4_re3-wp.md) | 文件类型伪装或假 exe 应先做 magic/文本首检，避免把明文 artifact 当二进制逆向。 | [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md) |
| [HGAME2026-看不懂的华容道-wp](../raw/reverse/HGAME2026-看不懂的华容道-wp.md) | VMP 包裹的华容道状态反馈可转成约束和 BFS，先恢复棋盘、碰撞指纹和最短路径。 | [vm-z3-sandbox-and-game-basics.md](vm-z3-sandbox-and-game-basics.md) |
| [HGAME2026-衔尾蛇-wp](../raw/reverse/HGAME2026-衔尾蛇-wp.md) | Spring Boot JAR 动态替换真实风控引擎并进入自定义 VM，先恢复 JVM 业务层和隐藏 VM 调用。 | [go-rust-jvm-and-cpp-reversing.md](go-rust-jvm-and-cpp-reversing.md) |
| [HGAME2026-androuge-wp](../raw/reverse/HGAME2026-androuge-wp.md) | 移动端、游戏或运行时平台题，先分清资源、脚本、native 和存档校验边界。 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md) |
| [HGAME2026-marionette-wp](../raw/reverse/HGAME2026-marionette-wp.md) | VM/解释器/混淆变换是主障碍，先定位 dispatch、状态寄存器、opcode 和真实比较点。 | [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md) |
| [HGAME2026-noncesense-wp](../raw/reverse/HGAME2026-noncesense-wp.md) | Windows client/driver 通过 IOCTL 交换 nonce、HMAC token 和 AES blob，先确认设备协议和驱动侧 key 派生。 | [windows-kernel-ioctl-hidden-feedback-maze.md](windows-kernel-ioctl-hidden-feedback-maze.md) |
| [HGAME2026-pvz-wp](../raw/reverse/HGAME2026-pvz-wp.md) | Java/JAR 游戏用植物阵型 hash 解密 flag，先定位 FlagScreen/GameScreen 并爆破小范围状态。 | [go-rust-jvm-and-cpp-reversing.md](go-rust-jvm-and-cpp-reversing.md) |
| [HGAME2026-signal-storm-wp](../raw/reverse/HGAME2026-signal-storm-wp.md) | 壳、trace、自解密或运行时阶段，先抓明文 buffer、key 和 I/O 边界。 | [signal-trace-and-packed-anti-analysis.md](signal-trace-and-packed-anti-analysis.md) |
| [HGAME2026-vidarchall-wp](../raw/reverse/HGAME2026-vidarchall-wp.md) | Android zygote preload、isolatedProcess 和 native 埋点共同影响 XXTEA 参数，先确认多进程运行时状态。 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md) |
| [LilacCTF2026-c-plus-plus-plus-plus-wp](../raw/reverse/LilacCTF2026-c-plus-plus-plus-plus-wp.md) | 语言运行时或复杂 C/C++ 抽象影响定位，先恢复符号、类型和真实业务函数。 | [go-rust-jvm-and-cpp-reversing.md](go-rust-jvm-and-cpp-reversing.md) |
| [LilacCTF2026-ezpython-wp](../raw/reverse/LilacCTF2026-ezpython-wp.md) | Python 字节码、Cython 或嵌入式 Python，先恢复调用边界、dump 对象和反编译路径。 | [python-bytecode-esolangs-and-uefi.md](python-bytecode-esolangs-and-uefi.md) |
| [LilacCTF2026-justrom-wp](../raw/reverse/LilacCTF2026-justrom-wp.md) | SPARC 32-bit big-endian ROM 和内存映射寄存器是主边界，先确认基址、COMMAND/INPUT/OUTPUT 语义和 ChaCha-like 校验。 | [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md) |
| [LilacCTF2026-kilogram-wp](../raw/reverse/LilacCTF2026-kilogram-wp.md) | 常规逆向首轮，先做载体、字符串、交叉引用、动态断点和最小解密脚本。 | [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md) |
| [LilacCTF2026-lambda-m-wp](../raw/reverse/LilacCTF2026-lambda-m-wp.md) | Python 字节码、Cython 或嵌入式 Python，先恢复调用边界、dump 对象和反编译路径。 | [python-bytecode-esolangs-and-uefi.md](python-bytecode-esolangs-and-uefi.md) |
| [LilacCTF2026-nineapple-wp](../raw/reverse/LilacCTF2026-nineapple-wp.md) | 移动端、游戏或运行时平台题，先分清资源、脚本、native 和存档校验边界。 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md) |
| [NCTF2026-鸡爪流高手-wp](../raw/reverse/NCTF2026-鸡爪流高手-wp.md) | 常规逆向首轮，先做载体、字符串、交叉引用、动态断点和最小解密脚本。 | [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md) |
| [NCTF2026-hook-my-secret-wp](../raw/reverse/NCTF2026-hook-my-secret-wp.md) | 运行时 hook、patch 或 oracle 可观测关键状态，先选断点和最小输入。 | [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md) |
| [NCTF2026-nomybank-wp](../raw/reverse/NCTF2026-nomybank-wp.md) | 常规逆向首轮，先做载体、字符串、交叉引用、动态断点和最小解密脚本。 | [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md) |
| [NCTF2026-pay-for-2048-wp](../raw/reverse/NCTF2026-pay-for-2048-wp.md) | 移动端、游戏或运行时平台题，先分清资源、脚本、native 和存档校验边界。 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md) |
| [NCTF2026-vm-encryptor-wp](../raw/reverse/NCTF2026-vm-encryptor-wp.md) | VM/解释器/混淆变换是主障碍，先定位 dispatch、状态寄存器、opcode 和真实比较点。 | [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md) |
| [RCTF2025-chaos-wp](../raw/reverse/RCTF2025-chaos-wp.md) | 低成本运行即可暴露结果时，先做安全运行和输出验证，再决定是否需要深入逆向。 | [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md) |
| [RCTF2025-chaos2-wp](../raw/reverse/RCTF2025-chaos2-wp.md) | 大量花指令、反调试和动态 key 修改掩护 RC4 解密，先清理 junk、跟构造函数和密钥更新点。 | [signal-trace-and-packed-anti-analysis.md](signal-trace-and-packed-anti-analysis.md) |
| [RCTF2025-onion-wp](../raw/reverse/RCTF2025-onion-wp.md) | VM/解释器/混淆变换是主障碍，先定位 dispatch、状态寄存器、opcode 和真实比较点。 | [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md) |
| [Spirit2026-5-cythonchecker-wp](../raw/reverse/Spirit2026-5-cythonchecker-wp.md) | Python 字节码、Cython 或嵌入式 Python，先恢复调用边界、dump 对象和反编译路径。 | [python-bytecode-esolangs-and-uefi.md](python-bytecode-esolangs-and-uefi.md) |
| [Spirit2026-5-im-a-human-wp](../raw/reverse/Spirit2026-5-im-a-human-wp.md) | 壳、trace、自解密或运行时阶段，先抓明文 buffer、key 和 I/O 边界。 | [signal-trace-and-packed-anti-analysis.md](signal-trace-and-packed-anti-analysis.md) |
| [Spirit2026-5-kernelmaze-wp](../raw/reverse/Spirit2026-5-kernelmaze-wp.md) | 驱动/内核接口隐藏反馈，先确认 IOCTL、用户态/内核态边界和可观测输出。 | [windows-kernel-ioctl-hidden-feedback-maze.md](windows-kernel-ioctl-hidden-feedback-maze.md) |
| [Spirit2026-5-link-start-wp](../raw/reverse/Spirit2026-5-link-start-wp.md) | 壳、trace、自解密或运行时阶段，先抓明文 buffer、key 和 I/O 边界。 | [signal-trace-and-packed-anti-analysis.md](signal-trace-and-packed-anti-analysis.md) |
| [Spirit2026-5-xxxtea-wp](../raw/reverse/Spirit2026-5-xxxtea-wp.md) | 程序最终先解出明文再与输入比较，先满足长度检查并在最终比较前断下读取明文。 | [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md) |
| [SU_easygalWP](../raw/reverse/SU_easygalWP.md) | 常规逆向首轮，先做载体、字符串、交叉引用、动态断点和最小解密脚本。 | [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md) |
| [SU_flumelWP](../raw/reverse/SU_flumelWP.md) | 语言运行时或复杂 C/C++ 抽象影响定位，先恢复符号、类型和真实业务函数。 | [go-rust-jvm-and-cpp-reversing.md](go-rust-jvm-and-cpp-reversing.md) |
| [SU_LockWP](../raw/reverse/SU_LockWP.md) | 复杂校验最终落到比较点，先断在比较/输出前恢复明文或中间 buffer。 | [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md) |
| [SU_MvsicPlayerWP](../raw/reverse/SU_MvsicPlayerWP.md) | 语言运行时或复杂 C/C++ 抽象影响定位，先恢复符号、类型和真实业务函数。 | [go-rust-jvm-and-cpp-reversing.md](go-rust-jvm-and-cpp-reversing.md) |
| [SU_old_binWP](../raw/reverse/SU_old_binWP.md) | 常规逆向首轮，先做载体、字符串、交叉引用、动态断点和最小解密脚本。 | [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md) |
| [SU_protocolWP](../raw/reverse/SU_protocolWP.md) | 自实现 HTTP/body 解码和私有二进制协议是首要边界，先复现解析流程、block 变换和固定目标比较。 | [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md) |
| [SU_RevirdWP](../raw/reverse/SU_RevirdWP.md) | 常规逆向首轮，先做载体、字符串、交叉引用、动态断点和最小解密脚本。 | [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md) |
| [SU_WestWP](../raw/reverse/SU_WestWP.md) | VM/解释器/混淆变换是主障碍，先定位 dispatch、状态寄存器、opcode 和真实比较点。 | [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md) |
| [VNCTF2026-delicious-obf-ez-maze-wp](../raw/reverse/VNCTF2026-delicious-obf-ez-maze-wp.md) | VM/解释器/混淆变换是主障碍，先定位 dispatch、状态寄存器、opcode 和真实比较点。 | [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md) |
| [VNCTF2026-login-wp](../raw/reverse/VNCTF2026-login-wp.md) | 常规逆向首轮，先做载体、字符串、交叉引用、动态断点和最小解密脚本。 | [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md) |
| [VNCTF2026-shadow-wp](../raw/reverse/VNCTF2026-shadow-wp.md) | 壳、trace、自解密或运行时阶段，先抓明文 buffer、key 和 I/O 边界。 | [signal-trace-and-packed-anti-analysis.md](signal-trace-and-packed-anti-analysis.md) |
| [D3CTF2019-ancient-game-v2-wp](../raw/reverse/D3CTF2019-ancient-game-v2-wp.md) | OISC/NAND 自定义 VM 实现数独约束，先抽取不跳 wrong 的控制流约束再交给 solver。 | [vm-z3-sandbox-and-game-basics.md](vm-z3-sandbox-and-game-basics.md) |
| [D3CTF2019-ch1pfs-wp](../raw/reverse/D3CTF2019-ch1pfs-wp.md) | 自定义文件系统镜像和 RC4 文件层加密，先恢复元数据结构和已知明文 keystream。 | [loader-vm-image-and-kernel-patterns.md](loader-vm-image-and-kernel-patterns.md) |
| [D3CTF2019-disappeared-memory-wp](../raw/reverse/D3CTF2019-disappeared-memory-wp.md) | Windows 10 compressed memory 导致关键页缺失，先从 dump/PTE/Store Manager 恢复数据页。 | [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md) |
| [D3CTF2019-easy-dongle-wp](../raw/reverse/D3CTF2019-easy-dongle-wp.md) | ELF 加密狗和 STM32 固件经 UART 协议协作，先还原固件加载地址、串口封包和 DES 参数。 | [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md) |
| [D3CTF2019-keygenme-wp](../raw/reverse/D3CTF2019-keygenme-wp.md) | Keygen 逻辑落到签名/曲线数学关系，先把校验式转成可求解的 DSA/ECDSA 约束。 | [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md) |
| [D3CTF2019-machine-wp](../raw/reverse/D3CTF2019-machine-wp.md) | Android/Frida 运行时平台是主边界，先确认 Java/native 调用和动态 hook 位置。 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md) |
| [D3CTF2019-simd-wp](../raw/reverse/D3CTF2019-simd-wp.md) | AVX2 SIMD 并行 SM4 校验，先理解 gather/shuffle 后的数据布局再按真实排列解密。 | [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md) |
| [D3CTF2021-ancient-wp](../raw/reverse/D3CTF2021-ancient-wp.md) | 算术编码和编译期字符串保护组合，先 dump 固定分布表和目标编码再反解输入。 | [self-decrypting-strings-and-lattice-patterns.md](self-decrypting-strings-and-lattice-patterns.md) |
| [D3CTF2021-baby-spear-wp](../raw/reverse/D3CTF2021-baby-spear-wp.md) | 隐藏 VBA 宏释放 PE 并用时间派生 AES key，先恢复 Office 宏流和 staged payload。 | [scripts-and-obfuscation.md](scripts-and-obfuscation.md) |
| [D3CTF2021-jumpjump-wp](../raw/reverse/D3CTF2021-jumpjump-wp.md) | 静态 ELF 用 setjmp/longjmp 拆分控制流，先识别异常式跳转再提取 magic 数组反推。 | [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md) |
| [D3CTF2021-no-name-wp](../raw/reverse/D3CTF2021-no-name-wp.md) | Android assets 加密 Java 校验代码由 native 返回 key，先恢复运行时真实验证接口。 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md) |
| [D3CTF2021-white-give-wp](../raw/reverse/D3CTF2021-white-give-wp.md) | LLVM pass 全局变量 AES、常数拆分和 MBA 表达式替换，先 dump 明文常量再还原校验流。 | [self-decrypting-strings-and-lattice-patterns.md](self-decrypting-strings-and-lattice-patterns.md) |
| [D3CTF2021-zigzag-encryptor-wp](../raw/reverse/D3CTF2021-zigzag-encryptor-wp.md) | Zigzag 图形编码叠加 LFSR 流密码，先还原图像排列再用已知明文恢复递推。 | [rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md) |
| [D3CTF2022-d3arm-wp](../raw/reverse/D3CTF2022-d3arm-wp.md) | ARM 架构/指令语义是主障碍，先确认加载基址、调用约定和平台特有指令。 | [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md) |
| [D3CTF2022-d3hotel-wp](../raw/reverse/D3CTF2022-d3hotel-wp.md) | Unity WebGL/IL2CPP/WASM 和 Lua 假 flag 组合，先建立 metadata、Lua、wasm 函数映射。 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md) |
| [D3CTF2022-d3mug-wp](../raw/reverse/D3CTF2022-d3mug-wp.md) | Unity3D Android 游戏可通过 Frida 调用 native update/get，先拆资源谱面并复用外部函数。 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md) |
| [D3CTF2022-d3re-wp](../raw/reverse/D3CTF2022-d3re-wp.md) | AES/自定义校验逻辑是主线，先定位真实比较点和可 dump 的中间明文/密文。 | [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md) |
| [D3CTF2022-d3thon-wp](../raw/reverse/D3CTF2022-d3thon-wp.md) | Cython 编译的 Python VM 信息泄露明显，先直接 import/调用模块枚举字节码语义。 | [python-bytecode-esolangs-and-uefi.md](python-bytecode-esolangs-and-uefi.md) |
| [D3CTF2022-d3w0w-wp](../raw/reverse/D3CTF2022-d3w0w-wp.md) | 32 位入口隐藏 WoW64 64 位逻辑，先 dump/按 64-bit 解析再还原 Masyu 路径约束。 | [windows-kernel-ioctl-hidden-feedback-maze.md](windows-kernel-ioctl-hidden-feedback-maze.md) |
| [D3CTF2023-d3hell-wp](../raw/reverse/D3CTF2023-d3hell-wp.md) | exe+dll 通过 DllMain 注入真假输入和反调试，先区分 TEA 解密真参数与 fake flag。 | [anti-analysis.md](anti-analysis.md) |
| [D3CTF2023-d3rc4-wp](../raw/reverse/D3CTF2023-d3rc4-wp.md) | RC4 或流密码恢复是主线，先确认 key 调度、明密文边界和 keystream 可复用性。 | [rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md) |
| [D3CTF2023-d3recover-wp](../raw/reverse/D3CTF2023-d3recover-wp.md) | Cython/Python 扩展可用带符号版本 BinDiff 迁移语义，先恢复 Pyx API 调用和约束。 | [python-bytecode-esolangs-and-uefi.md](python-bytecode-esolangs-and-uefi.md) |
| [D3CTF2023-d3sky-wp](../raw/reverse/D3CTF2023-d3sky-wp.md) | TLS 反调试和 RC4 加密 opcode 的自修改 VM，先恢复正确异常路径和滑动 XOR 关系。 | [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md) |
| [D3CTF2023-d3syscall-wp](../raw/reverse/D3CTF2023-d3syscall-wp.md) | 内核模块把保留 syscall 映射成 VM 指令，先 strace 参数并恢复 syscall 到 opcode 的语义。 | [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md) |
| [D3CTF2023-d3tetris-wp](../raw/reverse/D3CTF2023-d3tetris-wp.md) | Android/Java 游戏和 native 校验混合，先拆资源/运行时调用再恢复游戏状态或算法。 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md) |
| [D3CTF2025-aliceinpuzzle-wp](../raw/reverse/D3CTF2025-aliceinpuzzle-wp.md) | 父进程 ptrace、实时信号、SMC 和 memfd/fexecve 串联，先分离反分析外壳和内层 puzzle。 | [anti-analysis.md](anti-analysis.md) |
| [D3CTF2025-d3kernel-wp](../raw/reverse/D3CTF2025-d3kernel-wp.md) | Windows kernel/userland 混合逆向，先绕过 R3/R0 反调试并定位 DeviceIoControl VM。 | [windows-kernel-ioctl-hidden-feedback-maze.md](windows-kernel-ioctl-hidden-feedback-maze.md) |
| [D3CTF2025-d3piano-wp](../raw/reverse/D3CTF2025-d3piano-wp.md) | Android 钢琴 app 的 native hook 链和 Salsa20/LZW 音符序列，先识别 Frida/gumpp 篡改点。 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md) |
| [D3CTF2025-d3rpg-revenge-wp](../raw/reverse/D3CTF2025-d3rpg-revenge-wp.md) | RPG Maker/RGSS 资源包魔改和 Ruby 脚本明文内存，先脱壳或搜索脚本恢复校验。 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md) |
| [D3CTF2025-locked-door-wp](../raw/reverse/D3CTF2025-locked-door-wp.md) | VMP 壳和反调试后还有 RSA 签名门，先脱壳定位 OEP，再替换公钥或重签 key。 | [packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md) |

## 原始资料
- [reverse-first-pass-workflow-and-debugging.md](../raw/reverse/reverse-first-pass-workflow-and-debugging.md)
