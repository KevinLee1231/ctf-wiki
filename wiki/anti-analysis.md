---
type: family
tags: [reverse, family, anti-debug, anti-vm, anti-dbi, anti-disassembly]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/anti-analysis.md
  - ../raw/reverse/WMCTF2025-catfriend-wp.md
  - ../raw/reverse/WMCTF2025-videoplayer-wp.md
  - ../raw/reverse/WMCTF2025-want2become-magicalgirl-wp.md
updated: 2026-06-12
---

# Anti-Analysis Techniques & Bypasses

## 作用边界

本页是 Reverse 反分析 family 页，负责在调试、trace、hook、dump、符号恢复和自动化分析被干扰时做二级分流。它覆盖 Linux/Windows/macOS/Android 的 anti-debug、anti-VM、anti-sandbox、anti-DBI、代码完整性、自校验、反反汇编、MBA 和混淆控制流。

它不是单一 technique。遇到反分析时，首轮目标通常不是完整还原所有保护，而是找到最短稳定分析路线：patch 检测点、伪造环境、换调试器、提前 dump、动态 hook、trace 真实比较点，或者绕过保护后转入算法恢复。

## 共同识别信号

- 一上调试器就退出、卡死、走假分支，或输出与非调试运行不一致。
- 出现 `ptrace`、`/proc/self/status`、`rdtsc`、`alarm`、`SIGTRAP`、`IsDebuggerPresent`、`NtQueryInformationProcess`、TLS callback、PEB、DR register、Frida/Pin/DynamoRIO 检测。
- 静态视图中存在 junk bytes、overlapping instruction、opaque predicate、control-flow flattening、MBA、自校验或代码段 CRC。
- Android/Flutter/Java/native 混合样本检测 hook、Frida 端口、debuggable 状态、libart hook 或 Java 方法语义。
- 保护绕过后才出现真实解密、VM dispatch、比较点或 flag 校验。

## 最小证据

- 能定位至少一个检测点或分叉点，包括调用 API、syscall、信号、时间源、环境读取或自校验地址。
- 能比较正常运行和受控运行的差异：退出码、日志、分支、时间、信号、内存页或解密结果。
- 能说明绕过方式属于 patch、hook、环境伪造、调试器切换、提前 dump 还是动态 trace。
- 能确认绕过后下一步应进入哪个主线：壳/VM、字符串解密、比较点恢复、移动端 runtime 或工具页。

## 首轮路由

| 证据形态 | 首轮判断 | 下一跳 |
|---|---|---|
| Linux `ptrace`、`/proc`、`alarm`、signal、syscall 检测 | 先用 patch/hook/namespace/GDB signal policy 绕过单点，再确认是否还有第二层检测 | [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md) |
| `rdtsc`、`clock_gettime`、sleep 或时间阈值 | 先固定时间源或改比较阈值，避免 trace 工具改变行为 | [signal-trace-and-packed-anti-analysis.md](signal-trace-and-packed-anti-analysis.md) |
| Windows PEB、NtQuery、TLS callback、heap flag、DR register、INT3 scan | 先选择 x64dbg/ScyllaHide/TitanHide/VEH 或手工 patch，再找 OEP、真实入口和比较点 | [disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md), [packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md) |
| Anti-VM、硬件指纹、资源检查、注册表/文件痕迹 | 先判断能否伪造环境；不能稳定伪造时改用静态 patch 或真机/裸机采样 | [mobile-firmware-kernel-and-game-re.md](mobile-firmware-kernel-and-game-re.md) |
| Frida、Pin、DynamoRIO、Java hook 或 libart self-hook 检测 | 先调整注入时机和 hook 点，必要时改 native 检测逻辑，再恢复真实算法 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md) |
| 自校验、代码段 CRC、反断点扫描 | 先找校验范围和读代码路径，避免在被校验区域直接下软件断点 | [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md) |
| junk bytes、overlap、opaque predicate、CFF、MBA | 先修反汇编边界和控制流，再把表达式化简成可执行校验模型 | [packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md), [self-decrypting-strings-and-lattice-patterns.md](self-decrypting-strings-and-lattice-patterns.md) |
| VMP/壳/多层保护混合 | 不追完整保护实现，优先找稳定 dump、OEP、导入修复和真实逻辑入口 | [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md) |

## 合并与拆分结论

本页应保留为 family。`ptrace`、PEB、TLS callback、Frida 检测、anti-VM、anti-DBI 和反反汇编的具体绕过方式不同，但首轮判断都围绕“如何恢复可信分析环境”。拆成大量小 technique 会导致入口碎片化；合并进 Reverse 首轮页又会让总入口过长。

后续只有当某类检测积累到多篇 WP 且形成稳定工具路线时，才单独拆出 technique，例如 `linux-ptrace-proc-antidebug.md` 或 `android-frida-detection-bypass.md`。

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [WMCTF2025-catfriend-wp](../raw/reverse/WMCTF2025-catfriend-wp.md) | Mach-O 目标只有简单 `ptrace` 反调试，真正主线是魔改 ChaCha20 和 VM xor；反调试应作为前置障碍快速跨过。 |
| [WMCTF2025-videoplayer-wp](../raw/reverse/WMCTF2025-videoplayer-wp.md) | VMP 程序可用 TitanHide 或 CE VEH 调试，断 `GetSystemTimeAsFileTime` 找 OEP，再 dump 后处理真实登录和解密逻辑。 |
| [WMCTF2025-want2become-magicalgirl-wp](../raw/reverse/WMCTF2025-want2become-magicalgirl-wp.md) | Java 层检测 Frida 端口、Flutter 检测 Java hook，native `nativeadd` 还 self-hook libart；检测绕过和真实算法恢复需要一起建模。 |

## 常见陷阱

- 把反分析当作主线全部逆完，反而耽误真实算法恢复。
- patch 检测点后没有复测正常路径，导致进入假成功或假失败分支。
- 在自校验区域下 INT3 软件断点，触发了新的反调试。
- Android/Flutter 题只 hook Java 层，忽略 native init 或 libart 侧检测。
- dump 出来后不修导入、重定位或运行时解密状态，导致后续静态分析仍然偏差。

## 关联技巧

- [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md)
- [packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md)
- [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md)
- [signal-trace-and-packed-anti-analysis.md](signal-trace-and-packed-anti-analysis.md)
- [self-decrypting-strings-and-lattice-patterns.md](self-decrypting-strings-and-lattice-patterns.md)
- [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)
- [disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md)
- [reverse-tooling.md](reverse-tooling.md)

## 原始资料

- [anti-analysis.md](../raw/reverse/anti-analysis.md)
- [WMCTF2025-catfriend-wp](../raw/reverse/WMCTF2025-catfriend-wp.md)
- [WMCTF2025-videoplayer-wp](../raw/reverse/WMCTF2025-videoplayer-wp.md)
- [WMCTF2025-want2become-magicalgirl-wp](../raw/reverse/WMCTF2025-want2become-magicalgirl-wp.md)
