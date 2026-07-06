---
type: tooling
tags: [reverse, tooling, instrumentation, symbolic-execution, debugger]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/frida-angr-lldb-and-x64dbg.md
updated: 2026-07-06
---

# Frida, angr, lldb and x64dbg

## 作用边界

本页记录逆向中“静态分析已经不够，但还不一定需要完整模拟”的工具选择。Frida 适合运行时 hook 和 patch，angr 适合把路径条件交给符号执行，lldb/x64dbg 适合特定平台调试。它们解决的是观察和控制执行的问题，不应替代基础反汇编，也不应被当成所有复杂逆向的默认入口。

## 稳定调用方式

- Frida：先确认进程、架构、模块加载时机和 hook 点，再用 `frida -f ./target -l hook.js --no-pause` 或移动端 attach；脚本里优先枚举模块/导出后再 patch 具体函数。
- angr：先从 Ghidra/r2/GDB 得到 `find`、`avoid`、输入长度和字符集约束，再写最小 `angr.Project(...).factory.full_init_state()` 路线；库函数、复杂校验和环境输入能 hook 就 hook。
- lldb/x64dbg：先固定平台、位数、ASLR/PIE 基址和断点地址；用条件断点或脚本化断点记录寄存器/内存，不把长单步当作分析结果。
- 调用前必须有静态或动态证据支撑 hook/符号执行目标；如果还没有入口和比较点，先回 [disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md)。

## 工具路由

| 工具 | 触发信号 | 优先解决的问题 | 失败后转向 |
|---|---|---|---|
| Frida | 程序可运行，但关键值在运行时生成；需要 hook 函数、改返回值、扫内存或绕过反调试 | 动态插桩、函数替换、Stalker trace、移动端 Java/native 桥接 | hook 点不稳定时先回到 GDB/r2/Ghidra 定位；系统环境依赖重时转 Qiling。 |
| r2frida | 已在 Radare2 中定位函数，希望直接联动运行时观察 | 把静态地址、符号和 Frida hook 串起来 | 若地址随机化或加载时机复杂，先用 Frida 枚举模块和导出。 |
| angr | 输入空间大、分支条件清晰、目标/避开地址明确 | 自动路径探索、约束恢复、hook 掉复杂库函数 | 路径爆炸时收紧输入长度/字符集，或改用动态 trace 找关键分支。 |
| lldb | Mach-O、iOS/macOS、LLVM 生态或题目明确给出 lldb 线索 | 平台调试、断点脚本、寄存器和内存观察 | 若是移动端 app，通常要和 Frida/Jailbreak 环境配合。 |
| x64dbg | Windows PE、本地 GUI 调试、反调试和 patch 验证 | 条件断点、内存补丁、导入/异常/线程观察 | 反调试强时转 [anti-analysis.md](anti-analysis.md)，或先做静态 patch。 |

## 使用原则

- 先有地址、符号、导入、字符串或比较点，再写 hook；没有定位依据的全局 hook 很容易制造噪声。
- Frida 适合快速验证“如果这个函数返回 X 会怎样”；angr 适合回答“是否存在一条路径让状态满足 Y”。
- angr 的输入要尽量小，库函数要能 hook 就 hook，目标状态和避开状态要先从静态/动态证据中抽出来。
- 调试器用于确认事实，插桩用于改写事实；不要用长 trace 代替对关键状态的建模。

## 常见失败信号

- Frida hook 没触发：模块未加载、符号名错、地址受 ASLR 影响、native/Java 层选错，先枚举模块和导出。
- hook 触发但结果不变：patch 的不是真实校验点，回到比较点或错误分支重新定位。
- angr 卡死或内存暴涨：路径爆炸、输入过宽、外部函数未 hook，先缩短输入和约束字符集。
- x64dbg/lldb 单步后逻辑变化：反调试、时间检测或异常流参与校验，转 [anti-analysis.md](anti-analysis.md)。
- 移动端 Frida 连接失败：server/ABI/API level 不匹配，先确认进程、架构和权限，再排查脚本。

## 关联页面

- [disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md)
- [qiling-triton-pin-and-ldpreload.md](qiling-triton-pin-and-ldpreload.md)
- [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md)
- [anti-analysis.md](anti-analysis.md)
- [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)
- [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md)

## 原始资料

- [frida-angr-lldb-and-x64dbg.md](../raw/reverse/frida-angr-lldb-and-x64dbg.md)
