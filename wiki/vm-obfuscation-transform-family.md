---
type: family
tags: [reverse, family, vm, obfuscation, bytecode, smc, anti-debug, tracing]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/vm-obfuscation-transform-patterns.md
updated: 2026-07-28
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

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| 自定义 VM/WASM 需恢复 opcode、状态布局和 CFG | [custom-vm-and-wasm-state-lifting.md](custom-vm-and-wasm-state-lifting.md) |
| 动态生成/SMC/短暂明文适合 hook、trace 和快照 | [trace-hook-and-state-snapshot-reconstruction.md](trace-hook-and-state-snapshot-reconstruction.md) |
| VM/验证器状态可提升为 BitVec/路径约束 | [constraint-solver-and-symbolic-state-recovery.md](constraint-solver-and-symbolic-state-recovery.md) |

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
- [cython-and-python-extension-checker-recovery.md](cython-and-python-extension-checker-recovery.md)
- [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md)
- [go-rust-jvm-and-cpp-reversing.md](go-rust-jvm-and-cpp-reversing.md)
- [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md)
- [mobile-firmware-kernel-and-game-re.md](mobile-firmware-kernel-and-game-re.md)
- [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md)
- [windows-kernel-ioctl-hidden-feedback-maze.md](windows-kernel-ioctl-hidden-feedback-maze.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [ACTF2026-计算机系统贯通实验-wp](../raw/reverse/ACTF2026-计算机系统贯通实验-wp.md) | xlsx 公式实现 RISC-V 单周期 CPU；先解包 XML 恢复 ROM/RAM/MMIO、指令解码和输出语义，再拆四段校验。 |
| [ACTF2026-virtualnpu-wp](../raw/reverse/ACTF2026-virtualnpu-wp.md) | CUDA fatbin 中宿主先解出 NPU bytecode；提取 `MOV_IMM` 比较常量后逆 RC4 drop-512 和多层 S-box/XOR 变换。 |
| [D3CTF2023-d3sky-wp](../raw/reverse/D3CTF2023-d3sky-wp.md) | TLS 反调试和 RC4 加密 opcode 的自修改 VM，先恢复正确异常路径和滑动 XOR 关系。 |
| [D3CTF2023-d3syscall-wp](../raw/reverse/D3CTF2023-d3syscall-wp.md) | 内核模块把保留 syscall 映射成 VM 指令，先 strace 参数并恢复 syscall 到 opcode 的语义。 |
| [HGAME2026-androuge-wp](../raw/reverse/HGAME2026-androuge-wp.md) | APK 释放魔改 Lua 5.4 VM 与加密 `game` bytecode；先还原 XOR 载入层和 opcode 位域，再提密文数组与 seed。 |
| [NCTF2026-vm-encryptor-wp](../raw/reverse/NCTF2026-vm-encryptor-wp.md) | 先写自定义 VM disassembler 理清 opcode；真实算法是循环位移/XOR 后进魔改 Base64，再整体 XOR。 |
| [RCTF2025-onion-wp](../raw/reverse/RCTF2025-onion-wp.md) | 自定义 VM 有 PC/HIPC/LOTAG/HITAG/虚拟栈和 50 个 64-bit 输入；先实现反汇编/解释器，再把每个 check 自动逆算。 |
| [SUCTF2026-MvsicPlayerWP](../raw/reverse/SUCTF2026-MvsicPlayerWP.md) | Electron 音乐播放器先解析 `.su_mv` payload，再由 native `.node` 对 WAV 分支执行 VM bytecode 加密；目标是恢复原 WAV MD5。 |
| [ACTF2025-unstoppable-wp](../raw/reverse/ACTF2025-unstoppable-wp.md) | 本题把不可判定的停机问题限制成了已有严格上界的有限分类问题。逆向工作的最小目标是恢复 30 字节转移表和输入契约，不必为了“反混淆完整”而清理所有函数。拿到 VM 后，以 Coq-BB5 的精确步数界进行确定性模拟是最可靠方案；补丁加进程超时可以省逆向工作，但必须处理墙钟阈值带来的误判风险。 |
| [D3CTF2024-modern-legacy-wp](../raw/reverse/D3CTF2024-modern-legacy-wp.md) | 本题的关键是从 Rust 运行时入口、固定步长取指、PC 上界和设备 vtable 四组证据确认 MIX VM，而不是在巨大 Rust 反编译结果中逐函数硬追。动态转储初始化后的 `mem[4000]` 可以可靠提取字节码，但仍要注意 `STJ` 会在运行时修改返回跳转，从而改变密钥数据。 |
| [D3CTF2024-RandomVM-wp](../raw/reverse/D3CTF2024-RandomVM-wp.md) | 逆向重点是把随机化的基本块调度与真实 VM 数据流分开。`rand()` 交叉引用、系统调用块和会修改 `flagptr/slotptr` 的块是恢复控制流的有效锚点；重复的 `0-T` 只影响分支选择，不应原样堆进最终 WP。 |
| [MoeCTF2021-baby-bc-wp](../raw/reverse/MoeCTF2021-baby-bc-wp.md) | 面对 LLVM IR，先从字符串常量、比较位置和循环边界恢复高层数据流，通常比顺着 SSA 变量逐条追踪更高效。本题还需要特别注意表示层：`\5C` 是 LLVM 对反斜杠字节的转义，而目标串中的任意一个字符缺失都会破坏编码长度关系。识别出“RC4 加密后再做类 Base64 编码”后，两层操作都可以直接逆转。 |
| [MoeCTF2024-特工luo-深入敌营-wp](../raw/reverse/MoeCTF2024-特工luo-深入敌营-wp.md) | 自定义 VM 逆向应先恢复状态布局和每条 opcode 的精确宽度，再做符号执行。Z3 只是约束求解器，真正容易出错的是虚拟栈的字节序、弹栈长度、布尔归一化和条件跳转；先写具体值解释器与原程序逐步对拍，再替换成位向量最稳妥。 |
| [RCTF2025-determinism-wp](../raw/reverse/RCTF2025-determinism-wp.md) | 本题表面是海量动态解密代码，真正结构却很规整：Code 是“带权节点 + 字符约束”，Edge 是“状态位 + 二叉后继”。先离线解密并抽象图结构，再用两次 DFS 确定唯一记录区间，最后只把选中路径上的约束交给 Z3，能避免对数万函数逐个求解。实现时最容易出错的是把路径数字当数值而忘记程序传入的是 ASCII 字节，以及在 Z3 中丢失原始位宽、逻辑右移和低位截断。 |
| [RCTF2025-revme-wp](../raw/reverse/RCTF2025-revme-wp.md) | 本题不需要从 GHC 运行时海量样板中逐函数硬啃。定位到 parser、常量表和纯函数校验后，可以按可逆性分层处理：分段滚动哈希用中间相遇，超递增背包用贪心，镜像异或用置换后的成对约束，最后以位置校验消歧。尤其要防止两个直觉错误：哈希命中不代表分段唯一，`??` 也不是尚未求出的字符。 |
| [SekaiCTF2026-mikuprotect-wp](../raw/reverse/SekaiCTF2026-mikuprotect-wp.md) | 这类受保护程序不必先完整反编译整个虚拟机。动态轨迹负责回答“哪些 tape 读取最终形成目标常量”，短距离符号执行负责求新编码，二次模拟负责证明执行轨迹未漂移。尤其要区分“p-code 所在节”与“运行时确实作为 p-code 读取的字节”；服务端采用的是后者，单看节区边界不足以保证补丁合法。 |
| [SekaiCTF2026-nevm-wp](../raw/reverse/SekaiCTF2026-nevm-wp.md) | 本题的可靠性来自多层验证，而不是一次反编译后“看起来像某种密码”。面对超大 VM 保护程序，可以用动态轨迹限定真实执行面，用静态符号提升恢复输入无关的语义，再让编译器消除已经证明无效的垃圾。最终必须用随机输入比较 LLVM 与真实 EVM，并用目标明文正向重加密；两类检查分别防止单轨迹过拟合和参数识别错误。 |
| [SekaiCTF2026-Untitled-Encore-wp](../raw/reverse/SekaiCTF2026-Untitled-Encore-wp.md) | 关键是先把载体格式、内层指令、eBPF 一致性校验和 C++ 最终层拆开。四字节 VM 的 `NOTE` 操作已经把每个音符限制到很小的合法域，递归枚举加状态剪枝比搭建通用 VM 反编译器更直接；但只解出 chart 仍不够，还应重建 calldata，让真实 eBPF 返回值与内层 `KEY` 一致，并通过 C++ chart 统计后再解密。 |
| [UMDCTF2017-yarv-abuse-wp](../raw/reverse/UMDCTF2017-yarv-abuse-wp.md) | 字节码版本不兼容时，不必强行寻找完全相同的旧运行时。Marshal 数据仍保留了操作数、方法名、常量和调用顺序；只解释影响输出的少量指令，就能还原程序。这里的核心状态是递增的 `$LOL`，其余 VM 元数据都不是求解所必需。 |
| [UMDCTF2018-speed-demon-wp](../raw/reverse/UMDCTF2018-speed-demon-wp.md) | “加速 Ruby”是对预编译 VM 字节码的提示。遇到动态拼接、Fiddle 和 `rb_iseq_load` 时，不必先完全去混淆所有包装代码；先识别序列化格式，再反汇编或读取同仓库的原始指令序列，能够直接恢复常量和控制流。 |
| [UMDCTF2023-ep-815-wp](../raw/reverse/UMDCTF2023-ep-815-wp.md) | 识别无操作数指令中的未使用低 12 位，把不会改变程序语义的指令填充还原为连续隐藏字节；机器码能正常执行，但同一 opcode 的低位在不同位置呈现高熵、与规范汇编结果不一致时，应检查 instruction padding covert channel。 |
| [UMDCTF2023-i-heart-wasm-wp](../raw/reverse/UMDCTF2023-i-heart-wasm-wp.md) | WASM 的代码段、数据段和自定义段都应检查，不能只看导出函数；`WebAssembly.Module.customSections()` 按段名取内容，返回值是 `ArrayBuffer` 数组，需要再用 `TextDecoder` 解码。 |
| [UMDCTF2023-pokeptcha-wp](../raw/reverse/UMDCTF2023-pokeptcha-wp.md) | 图片提示来自经典的“从上方看到的胖丁”桥段，但只能确定语义，无法猜出精确大小写和 leetspeak；最终答案必须从校验逻辑恢复；第一层 Base64 和异或只是字节码封装，真正逻辑由运行时 `eval` 生成的 VM 处理器完成。 |
| [UMDCTF2023-well-connected-wp](../raw/reverse/UMDCTF2023-well-connected-wp.md) | 先用调用图的异常出度定位编码入口，再分析少量重复的小函数语义；大量随机函数名和相互调用是图结构噪声；无需逐个完整重命名。 |
| [UMDCTF2024-reality-wp](../raw/reverse/UMDCTF2024-reality-wp.md) | 这题的外壳是浮点地址虚拟机，内层则是矩阵分解校验。还原出“正交、上三角、乘积”三类约束后，决定性问题不是 QR 算法，而是把 `error < epsilon` 错当成了 `abs(error) < epsilon`。全零矩阵利用负残差通过单边比较，直接把复杂的数值求解降为 800 次发送 `0`。同时，公开仓库的发布二进制与当前生成源码规模不同，复现时必须先确认真实输入次数。 |
| [UMDCTF2024-typecheck-wp](../raw/reverse/UMDCTF2024-typecheck-wp.md) | 模板元编程只是执行载体，底层校验仍是一组普通线性约束。先把模板偏特化翻译成小型指令集，再用符号系数向量执行 VM，就能自动生成矩阵。浮点线性求解足以给出候选，但必须回代原始整数方程，避免四舍五入掩盖错误。 |
| [UMDCTF2024-undecidable-wp](../raw/reverse/UMDCTF2024-undecidable-wp.md) | 题目的“不可判定”外观来自图灵机和错误分支中的干扰状态，但固定长度输入只经过有限、可恢复的状态图。识别出相邻异或后，校验器就是 $\operatorname{GF}(2)$ 上的满秩线性变换。最稳妥的做法是让单位矩阵经历同一变换来自动生成 $M$，而不是手推 368 个输出位的公式。 |
| [UMDCTF2025-cmsc351-wp](../raw/reverse/UMDCTF2025-cmsc351-wp.md) | 大量同构函数应先抽象为图：节点是函数地址或编号，边记录分支字符、目标函数和状态更新；再用 BFS、Dijkstra 或带状态搜索找到终点，并单独模拟路径参数变化。 |
| [UMDCTF2025-computational-subway-surfers-wp](../raw/reverse/UMDCTF2025-computational-subway-surfers-wp.md) | 先把 3.5 MB CSS 动画还原为 VM 指令，再归纳状态转移与曲线检查，恢复短离散对数约束，最后用低误差 beam search 和 GF(2) 逆矩阵解码；耗时的有界离散对数应与轻量后处理分开。 |
| [UMDCTF2026-oracle-wp](../raw/reverse/UMDCTF2026-oracle-wp.md) | 层数多不等于每层都难。先按数据边界逐层拆开 AES、MWC、TLV 和 VM，再利用验证轮函数全部可逆这一事实从目标状态倒推输入。每层魔数、长度、正向寄存器结果和 AES 填充都能作为局部验收点，避免错误一直传播到最后。 |
| [UMDCTF2026-roulette-wp](../raw/reverse/UMDCTF2026-roulette-wp.md) | 验证函数逐块给出了可逆关系：余数去混淆后用 CRT 恢复 key，VM 只生成异或掩码，密文数组因此直接泄露明文块。唯一需要顺序执行的是滚动状态；若字节序或某轮状态更新错误，后续所有块都会失真。 |
| [WMCTF2024-easyandroid-wp](../raw/reverse/WMCTF2024-easyandroid-wp.md) | 第一层障碍是 native 中隐藏 LuaJIT bytecode；第二层障碍是 LuaJIT opcode 映射被魔改，普通反编译器需要先修映射。 |

## 原始资料

- [vm-obfuscation-transform-patterns.md](../raw/reverse/vm-obfuscation-transform-patterns.md)
