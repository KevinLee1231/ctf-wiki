---
type: family
tags: [reverse, family, triage]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/reverse-first-pass-workflow-and-debugging.md
updated: 2026-06-12
---

# First-Pass Workflow and Debugging

## 作用边界

本页是 Reverse 方向首轮 family，只负责把附件、运行行为和初步观察分流到更具体的 family / technique / tooling。不要在这里堆 payload 或工具百科；真正的解法骨架应落到具体页面。

首轮目标是回答四件事：

1. 载体是什么：ELF/PE/Mach-O/APK/WASM/pyc/JAR/固件/驱动/游戏资源/脚本/异常格式。
2. 障碍在哪里：平台运行时、语言对象、壳/反调试、VM/解释器、最终比较点、文件/图像/硬件载体还是密码/数学关系。
3. 最小可观察状态是什么：字符串、导入、入口、比较点、trace、dump、opcode、资源、协议、key/cipher 或中间 buffer。
4. 最短验证路径是什么：直接抓明文、恢复约束、写 forward checker、动态 hook、patch 反分析、提取资源或转其它方向。

## 首轮路由

| 初始信号 | 先去哪里 | 判断重点 |
|---|---|---|
| 未知二进制、普通 ELF/PE、需要首检和基础调试 | [disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md) | 格式、架构、保护、入口、导入、字符串、PIE 基址和可运行性。 |
| Android/APK、Flutter、Unity/Godot、Electron/Node、游戏资源 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md) | 平台层、资源层、脚本层和 native 层谁才是真校验入口。 |
| Mach-O/iOS、固件、驱动、eBPF、游戏引擎、CAN | [mobile-firmware-kernel-and-game-re.md](mobile-firmware-kernel-and-game-re.md) | 运行环境、签名/entitlements、文件系统、架构、IOCTL/设备边界。 |
| Go/Rust/JVM/C++/Swift/Kotlin/D/Haskell/Cython | [go-rust-jvm-and-cpp-reversing.md](go-rust-jvm-and-cpp-reversing.md) | 语言运行时、对象布局、符号、类型、协程/异常和标准库模式。 |
| Python bytecode、Pyarmor、Nuitka、Brainfuck、UEFI、HarmonyOS ABC | [python-bytecode-esolangs-and-uefi.md](python-bytecode-esolangs-and-uefi.md) | 先恢复字节码/解释器/低频格式，再谈算法。 |
| 自定义 VM、dispatch loop、opcode、flattening、字节变换 | [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md) | handler、状态寄存器、opcode stream、trace 和 forward checker。 |
| 二阶段 loader、image/bitmap、kernel module、binfmt、shared library backdoor | [loader-vm-image-and-kernel-patterns.md](loader-vm-image-and-kernel-patterns.md) | 真实逻辑何时加载、映射、解密或被内核/loader 解释。 |
| ptrace、Frida 检测、TLS callback、anti-VM、时间/环境检测 | [anti-analysis.md](anti-analysis.md) | 先跨过检测，不要把反分析门槛误当主算法。 |
| 最终比较点可断、明文 buffer 可 dump | [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md) | 抓最终明文或中间 buffer，比完整逆向更短。 |
| hook/patch/oracle/trace/动态符号执行 | [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md)、[frida-angr-lldb-and-x64dbg.md](frida-angr-lldb-and-x64dbg.md) | 用运行时观察降维，而不是追完整伪代码。 |
| 字体、shader、BPF、MBR、legacy/side-channel 载体 | [font-shader-firmware-and-legacy-patterns.md](font-shader-firmware-and-legacy-patterns.md) | 先识别隐藏信息承载方式或可观测副作用。 |

## 失败后 pivot

- 静态伪代码不可读：回到汇编、trace、hook、dump、emulation 或最终比较点。
- 程序运行环境不稳定：先处理反调试、签名、权限、依赖、架构、端序和加载器。
- 看见复杂算法但能断最终比较：优先抓 buffer 或构造 oracle。
- 附件像图片/字体/固件/资源包：先做格式和资源层 triage，避免过早套普通 ELF/PE 工作流。
- 逆向后出现 DSA/ECC/RC4/LFSR/代数关系：转 crypto 页面，把 reverse 只当实现恢复层。

## 关联页面

- [reverse-tooling.md](reverse-tooling.md)
- [disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md)
- [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)
- [mobile-firmware-kernel-and-game-re.md](mobile-firmware-kernel-and-game-re.md)
- [go-rust-jvm-and-cpp-reversing.md](go-rust-jvm-and-cpp-reversing.md)
- [python-bytecode-esolangs-and-uefi.md](python-bytecode-esolangs-and-uefi.md)
- [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md)
- [loader-vm-image-and-kernel-patterns.md](loader-vm-image-and-kernel-patterns.md)
- [font-shader-firmware-and-legacy-patterns.md](font-shader-firmware-and-legacy-patterns.md)

## WP 案例沉淀

本节只抽取 raw WP 中可复用的识别信号和下一跳，不替代原始题解正文。

| Raw WP | 复用信号 | 下一跳 |
|---|---|---|
| [WMCTF2025-appfriend-wp](../raw/reverse/WMCTF2025-appfriend-wp.md) | Android Java 层很薄，关键 SM4 key/cipher 在 native so 和 init 段检测后；先定位 JNI，再提取常量和密文。 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md) |
| [WMCTF2025-catfriend-wp](../raw/reverse/WMCTF2025-catfriend-wp.md) | Mach-O 小程序含 ptrace 反调试和魔改 ChaCha20/VM xor，先恢复流密码轮函数，再按加解密同构跑一遍。 | [anti-analysis.md](anti-analysis.md)、[self-decrypting-strings-and-lattice-patterns.md](self-decrypting-strings-and-lattice-patterns.md) |
| [WMCTF2025-videoplayer-wp](../raw/reverse/WMCTF2025-videoplayer-wp.md) | Windows VMP/反调试后还有后门账户机器绑定，登录通过不够，必须复用目标机器信息 MD5 解密 `.mp0` 并 dump 返回 vector。 | [packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md)、[runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md) |
| [WMCTF2025-want2become-magicalgirl-wp](../raw/reverse/WMCTF2025-want2become-magicalgirl-wp.md) | Android Flutter + Java + native 混合，Frida 检测和 libart self-hook 会改变 Java XXTEA 真实语义，先用 smali trace 确认执行流。 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)、[anti-analysis.md](anti-analysis.md) |
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
