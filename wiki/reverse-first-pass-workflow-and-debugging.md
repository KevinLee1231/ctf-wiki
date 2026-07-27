---
type: family
tags: [reverse, family, triage]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/reverse-first-pass-workflow-and-debugging.md
updated: 2026-07-27
---

# First-Pass Workflow and Debugging

## 作用边界

本页是 Reverse 方向首轮 family，只负责把附件、运行行为和初步观察分流到更具体的 family / technique / tooling。不要在这里堆 payload 或工具百科；真正的解法骨架应落到具体页面。

首轮目标是回答四件事：

1. 载体是什么：ELF/PE/Mach-O/APK/WASM/pyc/JAR/固件/驱动/游戏资源/脚本/异常格式。
2. 障碍在哪里：平台运行时、语言对象、壳/反调试、VM/解释器、最终比较点、文件/图像/硬件载体还是密码/数学关系。
3. 最小可观察状态是什么：字符串、导入、入口、比较点、trace、dump、opcode、资源、协议、key/cipher 或中间 buffer。
4. 最短验证路径是什么：直接抓明文、恢复约束、写 forward checker、动态 hook、patch 反分析、提取资源或转其它方向。

## 识别信号

- 附件需要先理解可执行载体、运行环境、加载方式、资源格式或语言运行时，而不是直接套固定算法。
- 初始观察能落到入口、字符串、导入、资源、比较点、trace、dump、opcode、协议字段、key/cipher 或中间 buffer。
- 同一题可能同时出现壳、反调试、VM、平台 runtime、文件载体和 crypto 关系，需要先判断哪个是第一障碍。
- 已有运行结果或静态线索能支撑一个最短验证路径：抓明文、hook/patch、写 forward checker、dump 资源，或转 Crypto/Stego/Pwn 等对应专项。

## 最小证据

- 记录文件类型、架构、位数、端序、保护、入口、依赖和能否安全运行。
- 保存至少一组最小输入、运行反馈、错误状态或断点位置，说明目前可观察到哪个内部状态。
- 如果准备转具体 family，列出触发转向的证据：平台标记、语言 runtime、opcode stream、loader 行为、反分析检测、资源表或最终比较点。
- 如果准备转 Crypto/Stego/Pwn/Hardware/Mobile，说明 Reverse 已经恢复到哪个实现边界，避免把实现恢复和后续数学、载荷提取或利用步骤混在同一页。

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

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| loader/packer 运行时映射第二阶段，需要 dump 与修复 | [staged-loader-and-runtime-image-recovery.md](staged-loader-and-runtime-image-recovery.md) |
| Python/JVM/.NET/Go/Rust metadata/bytecode 可恢复语义 | [managed-runtime-metadata-and-bytecode-recovery.md](managed-runtime-metadata-and-bytecode-recovery.md) |
| 静态控制流失真，API/比较/阶段边界适合 trace/hook | [trace-hook-and-state-snapshot-reconstruction.md](trace-hook-and-state-snapshot-reconstruction.md) |

## 常见误判

- 看到复杂伪代码就先完整重写算法，忽略最终比较点、明文 buffer、trace 或资源 dump 这些更短路径。
- 把壳、反调试、签名、依赖缺失或运行环境问题误当作核心算法。
- 只按扩展名分流，没有用 magic、架构、加载器和运行时行为确认真实载体。
- 逆向恢复出密码学、隐藏载荷或约束关系后仍留在 Reverse 页内硬解，没有按决定性主障碍转到 Crypto、Stego 或其它可复用 technique。

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
| [ACTF2026-计算机系统贯通实验-wp](../raw/reverse/ACTF2026-计算机系统贯通实验-wp.md) | xlsx 公式实现 RISC-V 单周期 CPU；先解包 XML 恢复 ROM/RAM/MMIO、指令解码和输出语义，再拆四段校验。 | [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md)、[hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md) |
| [ACTF2026-abyssgate-wp](../raw/reverse/ACTF2026-abyssgate-wp.md) | 用户态 loader、二阶段 ELF、eBPF tracepoint 和内核模块共同校验，先合并 ioctl 参数改写和内核数据流。 | [loader-vm-image-and-kernel-patterns.md](loader-vm-image-and-kernel-patterns.md)、[mobile-firmware-kernel-and-game-re.md](mobile-firmware-kernel-and-game-re.md) |
| [ACTF2026-calc-my-point-wp](../raw/reverse/ACTF2026-calc-my-point-wp.md) | Rust/GMP/大整数表达式执行器是主边界，先恢复语言运行时对象、数值约束和 CRT 关系。 | [go-rust-jvm-and-cpp-reversing.md](go-rust-jvm-and-cpp-reversing.md) |
| [ACTF2026-flagchecker-wp](../raw/reverse/ACTF2026-flagchecker-wp.md) | LoongArch64 Go 静态程序破坏符号恢复，通过反射派生真实方法名；再把 shellcode SM4 层和 8 段 Feistel 环分开求逆。 | [go-rust-jvm-and-cpp-reversing.md](go-rust-jvm-and-cpp-reversing.md)、[hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md) |
| [ACTF2026-virtualnpu-wp](../raw/reverse/ACTF2026-virtualnpu-wp.md) | CUDA fatbin 中宿主先解出 NPU bytecode；提取 `MOV_IMM` 比较常量后逆 RC4 drop-512 和多层 S-box/XOR 变换。 | [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md)、[hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md)、[rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md) |
| [西湖论剑2023-BabyRE-wp](../raw/reverse/西湖论剑2023-BabyRE-wp.md) | 主函数只做自定义 base8 第一层，第二层藏在 `atexit` 回调；用输入尾部 6 位十进制 RC4 key 约束反解。 | [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md)、[rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md) |
| [西湖论剑2023-Berkeley-wp](../raw/reverse/西湖论剑2023-Berkeley-wp.md) | eBPF uprobe/uretprobe 把真实校验藏到内核侧 instrumentation，先提取 BPF 程序、hook 点和常量表。 | [loader-vm-image-and-kernel-patterns.md](loader-vm-image-and-kernel-patterns.md)、[mobile-firmware-kernel-and-game-re.md](mobile-firmware-kernel-and-game-re.md) |
| [西湖论剑2023-Dual-Personality-wp](../raw/reverse/西湖论剑2023-Dual-Personality-wp.md) | PE32 运行时 patch far jump 到 WoW64 `0x33` 代码段；必须按 64 位模式重反汇编后再逆 rolling add/xor。 | [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md)、[disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md) |
| [西湖论剑2023-EasyVT-wp](../raw/reverse/西湖论剑2023-EasyVT-wp.md) | `EasyVT.sys` 模拟 VT-x，驱动 VM-exit handler 只是调度壳；核心校验是 TEA 变体和 RC4，优先静态恢复 handler switch。 | [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md)、[block-mode-misuse-family.md](block-mode-misuse-family.md)、[rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md) |
| [TamilCTF2021-foolme-wp](../raw/reverse/TamilCTF2021-foolme-wp.md) | 早期 sleep/exit 和 decoy target 误导动态运行，先区分反分析门槛与真正静态校验逻辑。 | [anti-analysis.md](anti-analysis.md) |
| [TamilCTF2021-GoldDigger-wp](../raw/reverse/TamilCTF2021-GoldDigger-wp.md) | 验证函数形如 `input[index[i]] + const == target[i]`；提取置换表和目标数组后直接反填输入。 | [self-decrypting-strings-and-lattice-patterns.md](self-decrypting-strings-and-lattice-patterns.md) |
| [强网杯2019-JustRe-wp](../raw/reverse/强网杯2019-JustRe-wp.md) | Flag 分两段：前半段是 DWORD 加法/低字节 XOR 约束，后半段是固定 3DES-ECB 密文和 24 字节 key。 | [self-decrypting-strings-and-lattice-patterns.md](self-decrypting-strings-and-lattice-patterns.md)、[block-mode-misuse-family.md](block-mode-misuse-family.md) |
| [Xp0intCTF2017-MaybeNotStandrad-wp](../raw/reverse/Xp0intCTF2017-MaybeNotStandrad-wp.md) | 输入 45 字节、输出 60 字符且有 64 字符表，是标准 Base64 结构加非标准字母表；先还原表再解码。 | [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md)、[self-decrypting-strings-and-lattice-patterns.md](self-decrypting-strings-and-lattice-patterns.md) |
| [0xGame2022-week1-re2-wp](../raw/reverse/0xGame2022-week1-re2-wp.md) | 36 字符中间段经过带反馈的 `uint32` 乘加/XOR 哈希，目标数组固定；用 Z3 保留 32 位溢出语义求解。 | [self-decrypting-strings-and-lattice-patterns.md](self-decrypting-strings-and-lattice-patterns.md) |
| [0xGame2022-week2-re2-wp](../raw/reverse/0xGame2022-week2-re2-wp.md) | Go `math/big` 调用链是 RSA 校验；重点是提取 `n/c/e` 并注意 `SetString` 的 base 参数，密文是八进制。 | [go-rust-jvm-and-cpp-reversing.md](go-rust-jvm-and-cpp-reversing.md)、[rsa-attacks.md](rsa-attacks.md) |
| [HGAME2026-看不懂的华容道-wp](../raw/reverse/HGAME2026-看不懂的华容道-wp.md) | VMP 包裹的华容道状态反馈可转成约束和 BFS，先恢复棋盘、碰撞指纹和最短路径。 | [vm-z3-sandbox-and-game-basics.md](vm-z3-sandbox-and-game-basics.md) |
| [HGAME2026-衔尾蛇-wp](../raw/reverse/HGAME2026-衔尾蛇-wp.md) | Spring Boot JAR 动态替换真实风控引擎并进入自定义 VM，先恢复 JVM 业务层和隐藏 VM 调用。 | [go-rust-jvm-and-cpp-reversing.md](go-rust-jvm-and-cpp-reversing.md) |
| [HGAME2026-androuge-wp](../raw/reverse/HGAME2026-androuge-wp.md) | APK 释放魔改 Lua 5.4 VM 与加密 `game` bytecode；先还原 XOR 载入层和 opcode 位域，再提密文数组与 seed。 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)、[vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md) |
| [HGAME2026-marionette-wp](../raw/reverse/HGAME2026-marionette-wp.md) | 父进程用 `ptrace` 调度子进程 `int3; ret` block；hook 记录 RIP trace 后，还原输入差分和 AES-NI 校验。 | [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md)、[block-mode-misuse-family.md](block-mode-misuse-family.md) |
| [HGAME2026-noncesense-wp](../raw/reverse/HGAME2026-noncesense-wp.md) | Windows client/driver 通过 IOCTL 交换 nonce、HMAC token 和 AES blob，先确认设备协议和驱动侧 key 派生。 | [windows-kernel-ioctl-hidden-feedback-maze.md](windows-kernel-ioctl-hidden-feedback-maze.md) |
| [HGAME2026-pvz-wp](../raw/reverse/HGAME2026-pvz-wp.md) | Java/JAR 游戏用植物阵型 hash 解密 flag，先定位 FlagScreen/GameScreen 并爆破小范围状态。 | [go-rust-jvm-and-cpp-reversing.md](go-rust-jvm-and-cpp-reversing.md) |
| [HGAME2026-signal-storm-wp](../raw/reverse/HGAME2026-signal-storm-wp.md) | SIGSEGV/SIGTRAP/SIGFPE handler 改 RC4 状态，`TracerPid` 混入 key；先 patch 反调试或复现 handler 后断 `memcmp`。 | [anti-analysis.md](anti-analysis.md)、[rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md)、[signal-trace-and-packed-anti-analysis.md](signal-trace-and-packed-anti-analysis.md) |
| [HGAME2026-vidarchall-wp](../raw/reverse/HGAME2026-vidarchall-wp.md) | Android zygote preload、isolatedProcess 和 native 埋点共同影响 XXTEA 参数，先确认多进程运行时状态。 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md) |
| [LilacCTF2026-c-plus-plus-plus-plus-wp](../raw/reverse/LilacCTF2026-c-plus-plus-plus-plus-wp.md) | C# Native AOT 中 `XEngine` 是 Twofish-like 16 轮 Feistel；先按 RS/MDS、40 个 round key 和 whitening 恢复固定 key/IV。 | [go-rust-jvm-and-cpp-reversing.md](go-rust-jvm-and-cpp-reversing.md)、[block-mode-misuse-family.md](block-mode-misuse-family.md) |
| [LilacCTF2026-ezpython-wp](../raw/reverse/LilacCTF2026-ezpython-wp.md) | PyInstaller runtime hook 把自定义 `a85decode/b64decode` 写入 `builtins`，并动态改 `MX.__code__` 后才是 XXTEA 真实轮函数。 | [python-bytecode-esolangs-and-uefi.md](python-bytecode-esolangs-and-uefi.md)、[runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md) |
| [LilacCTF2026-justrom-wp](../raw/reverse/LilacCTF2026-justrom-wp.md) | SPARC 32-bit big-endian ROM 和内存映射寄存器是主边界，先确认基址、COMMAND/INPUT/OUTPUT 语义和 ChaCha-like 校验。 | [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md) |
| [LilacCTF2026-kilogram-wp](../raw/reverse/LilacCTF2026-kilogram-wp.md) | VMP 外壳只是前置障碍；输出文件保存 salt、被口令 key 保护的本地 key 和 RC4-like flag 密文。 | [packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md)、[rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md) |
| [LilacCTF2026-lambda-m-wp](../raw/reverse/LilacCTF2026-lambda-m-wp.md) | Lambda calculus/Scott encoding 只是表达层，真实语义是 GF(2^8) 有理函数插值；把 40 个点转成齐次线性方程组。 | [python-bytecode-esolangs-and-uefi.md](python-bytecode-esolangs-and-uefi.md)、[number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| [LilacCTF2026-nineapple-wp](../raw/reverse/LilacCTF2026-nineapple-wp.md) | iOS Swift 九宫格手势锁给出 `weight/target_all/map_list`；无需操作 UI，按加权和反查每个字符路径。 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)、[self-decrypting-strings-and-lattice-patterns.md](self-decrypting-strings-and-lattice-patterns.md) |
| [NCTF2026-鸡爪流高手-wp](../raw/reverse/NCTF2026-鸡爪流高手-wp.md) | 游戏协议和 ELO 结算是主线；低分保护检查更新前状态，结算写入无符号分数字段导致下溢登榜。 | [game-state-websocket-and-wasm.md](game-state-websocket-and-wasm.md)、[interpreter-jit-canary-and-integer-exploits.md](interpreter-jit-canary-and-integer-exploits.md) |
| [NCTF2026-hook-my-secret-wp](../raw/reverse/NCTF2026-hook-my-secret-wp.md) | 运行时 hook、patch 或 oracle 可观测关键状态，先选断点和最小输入。 | [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md) |
| [NCTF2026-nomybank-wp](../raw/reverse/NCTF2026-nomybank-wp.md) | Godot PCK 密钥、运行时解密 DLL、TLS callback hook 和 SMC 共同隐藏真实校验；先恢复资源和动态补丁链。 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)、[runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md) |
| [NCTF2026-pay-for-2048-wp](../raw/reverse/NCTF2026-pay-for-2048-wp.md) | Electron `app.asar` 中 JS bridge 调 WASM license 校验；先直接用 Node 调应用服务，再补 WASM digest/key 派生。 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)、[game-state-websocket-and-wasm.md](game-state-websocket-and-wasm.md) |
| [NCTF2026-vm-encryptor-wp](../raw/reverse/NCTF2026-vm-encryptor-wp.md) | 先写自定义 VM disassembler 理清 opcode；真实算法是循环位移/XOR 后进魔改 Base64，再整体 XOR。 | [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md)、[encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) |
| [RCTF2025-chaos-wp](../raw/reverse/RCTF2025-chaos-wp.md) | 运行即可输出结果的短题，先做格式、依赖和安全运行验证，再决定是否需要静态分析。 | [disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md) |
| [RCTF2025-chaos2-wp](../raw/reverse/RCTF2025-chaos2-wp.md) | 大量花指令、反调试和动态 key 修改掩护 RC4 解密，先清理 junk、跟构造函数和密钥更新点。 | [signal-trace-and-packed-anti-analysis.md](signal-trace-and-packed-anti-analysis.md) |
| [RCTF2025-onion-wp](../raw/reverse/RCTF2025-onion-wp.md) | 自定义 VM 有 PC/HIPC/LOTAG/HITAG/虚拟栈和 50 个 64-bit 输入；先实现反汇编/解释器，再把每个 check 自动逆算。 | [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md)、[vm-z3-sandbox-and-game-basics.md](vm-z3-sandbox-and-game-basics.md) |
| [Spirit2026-5-cythonchecker-wp](../raw/reverse/Spirit2026-5-cythonchecker-wp.md) | Loader 释放 `runtime_codec.pyd` 并传入 AES key；必须分析实际释放的 Cython 扩展和 ShiftRows 变体。 | [embedded-python-pyd-custom-aes.md](embedded-python-pyd-custom-aes.md)、[python-bytecode-esolangs-and-uefi.md](python-bytecode-esolangs-and-uefi.md) |
| [Spirit2026-5-im-a-human-wp](../raw/reverse/Spirit2026-5-im-a-human-wp.md) | 钓鱼页写剪贴板 `mshta`，`.mp3` 被当 HTA/JS 执行并拼接 PowerShell；按 JS charcode、hex、XOR 分阶段还原。 | [powershell-staged-payload-and-clipboard-phishing.md](powershell-staged-payload-and-clipboard-phishing.md)、[scripts-and-obfuscation.md](scripts-and-obfuscation.md) |
| [Spirit2026-5-kernelmaze-wp](../raw/reverse/Spirit2026-5-kernelmaze-wp.md) | 驱动/内核接口隐藏反馈，先确认 IOCTL、用户态/内核态边界和可观测输出。 | [windows-kernel-ioctl-hidden-feedback-maze.md](windows-kernel-ioctl-hidden-feedback-maze.md) |
| [Spirit2026-5-link-start-wp](../raw/reverse/Spirit2026-5-link-start-wp.md) | VMP client/server 两段输入：迷宫路径既是第一阶段答案又是 SMC XOR key，解密后函数生成 RC4 key 校验 flag。 | [vmp-client-server-smc-rc4-recovery.md](vmp-client-server-smc-rc4-recovery.md)、[packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md) |
| [Spirit2026-5-xxxtea-wp](../raw/reverse/Spirit2026-5-xxxtea-wp.md) | 程序最终先解出明文再与输入比较，先满足长度检查并在最终比较前断下读取明文。 | [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md) |
| [SUCTF2026-easygalWP](../raw/reverse/SUCTF2026-easygalWP.md) | Unity/IL2CPP 资源中反序列化 Story 节点；恢复 choice 的 weight/value/marker 后建模为带路径恢复的背包 DP。 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)、[vm-z3-sandbox-and-game-basics.md](vm-z3-sandbox-and-game-basics.md) |
| [SUCTF2026-flumelWP](../raw/reverse/SUCTF2026-flumelWP.md) | Flutter/Dart 输入先经 `Rc4Warp`，再由新版 `libjunk.so` 验证 Hermes bundle 并派生 AES-CBC key/IV；旧 placeholder 会误导。 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)、[rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md)、[block-mode-misuse-family.md](block-mode-misuse-family.md) |
| [SUCTF2026-LockWP](../raw/reverse/SUCTF2026-LockWP.md) | Inno Setup、Rust overlay、锁屏程序和内核驱动多层嵌套；最终 IOCTL 中 XXTEA-like dword 校验在驱动层。 | [loader-vm-image-and-kernel-patterns.md](loader-vm-image-and-kernel-patterns.md)、[windows-kernel-ioctl-hidden-feedback-maze.md](windows-kernel-ioctl-hidden-feedback-maze.md) |
| [SUCTF2026-MvsicPlayerWP](../raw/reverse/SUCTF2026-MvsicPlayerWP.md) | Electron 音乐播放器先解析 `.su_mv` payload，再由 native `.node` 对 WAV 分支执行 VM bytecode 加密；目标是恢复原 WAV MD5。 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)、[vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md) |
| [SUCTF2026-old_binWP](../raw/reverse/SUCTF2026-old_binWP.md) | 固件先 XOR 解包出 IMG0 容器，再修复损坏 ELF 和 TLS 布局；最后还原网络 challenge 与自定义块校验。 | [mobile-firmware-kernel-and-game-re.md](mobile-firmware-kernel-and-game-re.md)、[loader-vm-image-and-kernel-patterns.md](loader-vm-image-and-kernel-patterns.md) |
| [SUCTF2026-protocolWP](../raw/reverse/SUCTF2026-protocolWP.md) | HTTP 路由很薄，body 先 hex 再进私有协议帧；区分格式错、比较失败和 block 变换后再反推 payload。 | [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md)、[block-mode-misuse-family.md](block-mode-misuse-family.md) |
| [SUCTF2026-RevirdWP](../raw/reverse/SUCTF2026-RevirdWP.md) | 外层魔改 AES 解出第二阶段 EXE，随后通过 `\\.\Revird` 与驱动 op case 协同完成 AES-like 校验。 | [loader-vm-image-and-kernel-patterns.md](loader-vm-image-and-kernel-patterns.md)、[windows-kernel-ioctl-hidden-feedback-maze.md](windows-kernel-ioctl-hidden-feedback-maze.md) |
| [SUCTF2026-WestWP](../raw/reverse/SUCTF2026-WestWP.md) | 81 轮 permutation + dispatch table 更新共享状态；逆三个 rotate/add/xor helper 后，用 Unicorn 推进状态并约束求输入。 | [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md)、[vm-z3-sandbox-and-game-basics.md](vm-z3-sandbox-and-game-basics.md) |
| [VNCTF2026-delicious-obf-ez-maze-wp](../raw/reverse/VNCTF2026-delicious-obf-ez-maze-wp.md) | `delicious obf` 是 `call $5; push; ret` 控制流混淆、SMC 和反调试；`ez_maze` 是魔改 UPX/MFC 迷宫，脱壳后复刻固定种子 DFS 并 BFS。 | [packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md)、[anti-analysis.md](anti-analysis.md)、[vm-z3-sandbox-and-game-basics.md](vm-z3-sandbox-and-game-basics.md) |
| [VNCTF2026-login-wp](../raw/reverse/VNCTF2026-login-wp.md) | APK Java 层只组包，native so 完成魔改 AES、HTTP header 签名和 Frida/IDA 环境检测；可结合流量复算。 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)、[anti-analysis.md](anti-analysis.md) |
| [VNCTF2026-shadow-wp](../raw/reverse/VNCTF2026-shadow-wp.md) | 用户态迷宫只触发 `Sleep(0x32)`；真实校验在反射加载驱动、PTE hook、键盘记录和基于 `KeDelayExecutionThread` 参数解密的 shellcode。 | [loader-vm-image-and-kernel-patterns.md](loader-vm-image-and-kernel-patterns.md)、[windows-kernel-ioctl-hidden-feedback-maze.md](windows-kernel-ioctl-hidden-feedback-maze.md) |
| [D3CTF2019-ancient-game-v2-wp](../raw/reverse/D3CTF2019-ancient-game-v2-wp.md) | OISC/NAND 自定义 VM 实现数独约束，先抽取不跳 wrong 的控制流约束再交给 solver。 | [vm-z3-sandbox-and-game-basics.md](vm-z3-sandbox-and-game-basics.md) |
| [D3CTF2019-ch1pfs-wp](../raw/reverse/D3CTF2019-ch1pfs-wp.md) | 自定义文件系统镜像和 RC4 文件层加密，先恢复元数据结构和已知明文 keystream。 | [loader-vm-image-and-kernel-patterns.md](loader-vm-image-and-kernel-patterns.md) |
| [D3CTF2019-disappeared-memory-wp](../raw/reverse/D3CTF2019-disappeared-memory-wp.md) | Windows 10 compressed memory 导致关键页缺失，先从 dump/PTE/Store Manager 恢复数据页。 | [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md) |
| [D3CTF2019-easy-dongle-wp](../raw/reverse/D3CTF2019-easy-dongle-wp.md) | ELF 加密狗和 STM32 固件经 UART 协议协作，先还原固件加载地址、串口封包和 DES 参数。 | [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md) |
| [D3CTF2019-keygenme-wp](../raw/reverse/D3CTF2019-keygenme-wp.md) | Keygen 逻辑落到签名/曲线数学关系，先把校验式转成可求解的 DSA/ECDSA 约束。 | [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md) |
| [D3CTF2019-machine-wp](../raw/reverse/D3CTF2019-machine-wp.md) | Android/Frida 运行时平台是主边界，先确认 Java/native 调用和动态 hook 位置。 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md) |
| [D3CTF2019-simd-wp](../raw/reverse/D3CTF2019-simd-wp.md) | AVX2 SIMD 并行 SM4 校验，先理解 gather/shuffle 后的数据布局再按真实排列解密。 | [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md) |
| [D3CTF2021-ancient-wp](../raw/reverse/D3CTF2021-ancient-wp.md) | 算术编码和编译期字符串保护组合，先 dump 固定分布表和目标编码再反解输入。 | [self-decrypting-strings-and-lattice-patterns.md](self-decrypting-strings-and-lattice-patterns.md) |
| [D3CTF2021-baby-spear-wp](../raw/reverse/D3CTF2021-baby-spear-wp.md) | 隐藏 VBA 宏释放 PE 并用时间派生 AES key，先恢复 Office 宏流和 staged payload。 | [scripts-and-obfuscation.md](scripts-and-obfuscation.md) |
| [D3CTF2021-jumpjump-wp](../raw/reverse/D3CTF2021-jumpjump-wp.md) | 静态 ELF 用 `setjmp/longjmp` 拆分控制流；先把异常式跳转还原成条件分支，再提取 magic 数组反推。 | [disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md)、[compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md) |
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
| [0xGame2020-week3-tls-rc4-wp](../raw/reverse/0xGame2020-week3-tls-rc4-wp.md) | TLS callback 是 PE 正常执行链的一部分，逆向时应在入口点之外检查 TLS Directory。算法识别也不能只凭“结构像 RC4”就替换成库函数；本题保留 KSA 末尾 `j` 的细节会改变整条密钥流，必须逐条复现实际状态更新。 | 本页对应路线 |
| [0xGame2024-week3-Tea-wp](../raw/reverse/0xGame2024-week3-Tea-wp.md) | TEA 的识别特征包括两个 32 位半块、四字密钥、32 轮 Feistel 式更新和黄金分割常量。还原时不能只套标准算法，还要复现题目额外的数据布局：整段逆序、小端 `uint32_t` 解释，以及相隔 32 字节的两个有效块。 | 本页对应路线 |
| [0xGame2024-week3-The-Matrix-wp](../raw/reverse/0xGame2024-week3-The-Matrix-wp.md) | 矩阵逆向题通常有两层工作：先从指针访问和内存对齐恢复数据结构，再把程序循环翻译成线性代数。这里除了求 $K^{-1}B$，还必须处理密钥实际顺序以及 9 字节窗口每次只前进 6 字节的重叠关系；使用精确有理数运算可避免“浮点后手工四舍五入”带来的不确定性。 | 本页对应路线 |
| [0xGame2025-week2-Shuffle-Shuffle-wp](../raw/reverse/0xGame2025-week2-Shuffle-Shuffle-wp.md) | 固定随机洗牌本质上只是一个固定置换。若随机数运行库已知，可以重放 Fisher–Yates；若跨平台 `rand()` 不一致，则用等长、字符互异的 chosen plaintext 直接标记每个位置。本题必须保留 44 字节密文、探针输入和探针输出，三者足以在没有原程序和外部网站的情况下复现 flag。 | 本页对应路线 |
| [0xGame2025-week4-蔷薇地狱-wp](../raw/reverse/0xGame2025-week4-蔷薇地狱-wp.md) | 本题把文件加密器逆向、流量取证和弱 RSA 串在一起。看到随机生成的 key/IV、`send` 外传和 PCAP 中的长十进制整数，应沿数据流确认“字节序 → 大整数 → 模幂 → 十进制编码”的完整序列，而不是直接把抓包内容当作对称密钥。还要以实际 API 参数为准判断对称模式：存在 IV 并不必然代表 CBC，本题明确设置的是 AES-256-ECB，IV 只是被多余地保留在协议和命令行接口中。 | 本页对应路线 |
| [MoeCTF2021-ez-algorithm-revenge-wp](../raw/reverse/MoeCTF2021-ez-algorithm-revenge-wp.md) | 这题把逆向得到的数据生成逻辑与经典数塔动态规划组合在了一起。除了写出状态转移和回溯，最重要的检查点是伪随机数实现：固定种子只能保证同一种 `rand()` 实现内的序列可复现，不能跨运行库直接等价。遇到类似题目，应同时记录种子、调用顺序、取模方式和实际运行时。 | 本页对应路线 |
| [MoeCTF2024-moedaily-wp](../raw/reverse/MoeCTF2024-moedaily-wp.md) | 分析表格型逆向题不能只看最终比较常量，应沿跨工作表引用还原数据流。这里 `BITAND(..., 0xffffffff)` 明确了 32 位截断，列间引用揭示每次 16 轮的分段，而第二组公式以上一组输出为输入，证明 TEA 被完整执行了两遍。 | 本页对应路线 |
| [SekaiCTF2026-minions-in-16k-wp](../raw/reverse/SekaiCTF2026-minions-in-16k-wp.md) | 本题的交付不是固定 flag，而是可持续参赛的协议实现和战斗 bot。连上服务器只完成了一半：评分是零和 Elo，净负收益 bot 持续运行会主动掉分，达到高水位后停机也是合理策略。应先在离线模式验证协议和控制逻辑，再以本题独立 Elo/净胜率监控线上效果。由于主要障碍是从客户端恢复网络布局、定点数与输入语义，本篇从 `_unclassified` 调整到 `reverse`。 | 本页对应路线 |
| [UMDCTF2017-break-the-chain-wp](../raw/reverse/UMDCTF2017-break-the-chain-wp.md) | 链式校验仍然可以逐步求解，只要状态依赖是单向的。与其一次性暴力搜索 $95^{46}$ 个字符串，不如把“前一字符”作为已知状态，每轮只枚举 95 个可打印字符，并同时用两条位运算约束剪枝。 | 本页对应路线 |
| [UMDCTF2017-hash-me-not-wp](../raw/reverse/UMDCTF2017-hash-me-not-wp.md) | 哈希或校验和“不可逆”不等于无法恢复：若每次只处理一个字符，输入空间只有 95 个候选，预计算反查表就足够。判断攻击成本时必须看分块方式和输入熵，而不能只看函数名称。 | 本页对应路线 |
| [UMDCTF2018-lorem-mipsum-wp](../raw/reverse/UMDCTF2018-lorem-mipsum-wp.md) | 逆向密码工件时应把每个变换独立实现，并分别验证中间结果。源码与附件发生冲突时，可打印明文只是线索，官方摘要才是强校验；本题也说明不能因为函数出现在控制流中，就未经验证地宣称公开密文一定经历了该函数。 | 本页对应路线 |
| [UMDCTF2019-armageddon-wp](../raw/reverse/UMDCTF2019-armageddon-wp.md) | 这题的难点是大量分散的关系约束，而不是某个单独的复杂算法。把每个校验函数独立执行，可以避免整程序路径爆炸；再将成功路径上与输入有关的约束合并，求解会稳定得多。符号执行结果仍应通过长度、固定格式和官方摘要交叉验证。 | 本页对应路线 |
| [UMDCTF2019-easycrypt-wp](../raw/reverse/UMDCTF2019-easycrypt-wp.md) | “使用随机数”不等于不可逆：种子若完全由公开的文件长度决定，排列就可以确定性重放。面对自定义加密，应把流程拆成字节代换和位置置换两层，分别证明可逆，再按加密的逆序恢复。输出文件的 magic、可视内容和官方摘要构成三重校验。 | 本页对应路线 |
| [UMDCTF2019-simplecheck-wp](../raw/reverse/UMDCTF2019-simplecheck-wp.md) | 大量成对合法性检查常可抽象为图问题。本题把 key 转化为固定大小的 clique，升序搜索既去除了排列重复，也便于用剩余候选数剪枝。对于已能在原二进制上验证、但缺少服务端秘密的数据，应清楚区分“利用链已完成”和“历史 flag 不可恢复”。 | 本页对应路线 |
| [UMDCTF2021-what-is-memory-wp](../raw/reverse/UMDCTF2021-what-is-memory-wp.md) | 遇到“近似标准算法”时不能强行套标准解码器。先定位实现与标准的差异，再把差异建模成有限搜索。本题每轮只丢失一个 Base64 字符，候选数可通过字符集、长度和最终格式迅速收敛；系统 API 检查与编码算法无关。 | 本页对应路线 |
| [UMDCTF2021-willow-wp](../raw/reverse/UMDCTF2021-willow-wp.md) | 自定义树替换的重点不是给节点强行命名成 S 盒，而是忠实还原每一位决定的分支和叶值。映射非双射时，逆运算天然产生候选集合；链式异或提供逐位置约束，flag 格式负责最后消歧，不能随意选第一个逆像。 | 本页对应路线 |
| [UMDCTF2023-cleffas-surprise-wp](../raw/reverse/UMDCTF2023-cleffas-surprise-wp.md) | 识别“复制到可执行页再间接调用”的 shellcode 装载器，并直接跟踪 shellcode 的内存副作用；AArch64 的 `movk` 适合分段构造立即数；连续的移位值 $0,16,32,48$ 往往代表拼接字符串或常量。 | 本页对应路线 |
| [UMDCTF2023-lets-catch-up-wp](../raw/reverse/UMDCTF2023-lets-catch-up-wp.md) | 用 AES 固定 T-table 常量识别被改名、混淆的第三方实现，再与上游库做结构对照；前端传入的可见密钥是障眼法；真正决定结果的是密钥扩展后被覆盖的 `Ke`。 | 本页对应路线 |
| [WMCTF2022-seeee-wp](../raw/reverse/WMCTF2022-seeee-wp.md) | 先恢复字符置换表，再利用 ChaCha20 流密码“加密即解密”的性质反推第二段输入；程序要求 IV 和 ciphertext 两个参数，内部出现固定映射表和一段字节流常量时，应检查是否是置换层叠流密码层。 | 本页对应路线 |

## 原始资料
- [reverse-first-pass-workflow-and-debugging.md](../raw/reverse/reverse-first-pass-workflow-and-debugging.md)
