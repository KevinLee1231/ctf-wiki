---
type: technique
tags: [reverse, instrumentation, symbolic-execution, debugger]
skills: [ctf-reverse, ctf-mobile]
raw:
  - ../raw/reverse/frida-angr-lldb-and-x64dbg.md
updated: 2026-09-07
---

# Frida, angr, lldb and x64dbg

## 作用边界

本页记录逆向中“静态分析已经不够，但还不一定需要完整模拟”的工具选择。Frida 适合运行时 hook 和 patch，angr 适合把路径条件交给符号执行，lldb/x64dbg 适合特定平台调试。它们解决的是观察和控制执行的问题，不应替代基础反汇编，也不应被当成所有复杂逆向的默认入口。

本页只维护选择与分析方法。当前安装状态、版本、路径、环境、完整命令和工具故障统一读取 [reverse-tooling.md](reverse-tooling.md)。

## 稳定调用方式

- Frida：先确认进程、架构、模块加载时机和 hook 点，再选择 launch 或 attach 并加载最小 hook；脚本里优先枚举模块/导出后再 patch 具体函数，当前完整命令只从 `reverse-tooling.md` 读取。
- angr：先从 IDA Pro MCP/r2/GDB 得到 `find`、`avoid`、输入长度和字符集约束，再写最小 `angr.Project(...).factory.full_init_state()` 路线；库函数、复杂校验和环境输入能 hook 就 hook。
- lldb/x64dbg：先固定平台、位数、ASLR/PIE 基址和断点地址；用条件断点或脚本化断点记录寄存器/内存，不把长单步当作分析结果。
- 调用前必须有静态或动态证据支撑 hook/符号执行目标；如果还没有入口和比较点，先回 [disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md)。

## 工具路由

| 工具 | 触发信号 | 优先解决的问题 | 失败后转向 |
|---|---|---|---|
| Frida | 程序可运行，但关键值在运行时生成；需要 hook 函数、改返回值、扫内存或绕过反调试 | 动态插桩、函数替换、Stalker trace、移动端 Java/native 桥接 | hook 点不稳定时先回到 IDA Pro MCP/GDB/r2 定位；IDA 不可用时 Ghidra 也可作普通静态辅助，系统环境依赖重时转 Qiling。 |
| r2frida | 已在 Radare2 中定位函数，希望直接联动运行时观察 | 把静态地址、符号和 Frida hook 串起来 | 若地址随机化或加载时机复杂，先用 Frida 枚举模块和导出。 |
| angr | 输入空间大、分支条件清晰、目标/避开地址明确 | 自动路径探索、约束恢复、hook 掉复杂库函数 | 路径爆炸时收紧输入长度/字符集，或改用动态 trace 找关键分支。 |
| lldb | Mach-O、iOS/macOS、LLVM 生态或题目明确给出 lldb 线索 | 平台调试、断点脚本、寄存器和内存观察 | 若是移动端 app，通常要和 Frida/Jailbreak 环境配合。 |
| x64dbg | Windows PE、本地 GUI 调试、反调试和 patch 验证 | 条件断点、内存补丁、导入/异常/线程观察 | 反调试强时转 [anti-analysis.md](anti-analysis.md)，或先做静态 patch。 |

## 使用原则

### Windows 参数观测与 patch 验证

1. 先固定本次问题：哪一个输入在什么调用或比较点变成了什么值。记录模块基址、RVA、线程和函数原型；ASLR 或模块重载后重新计算地址。
2. 在函数入口观测参数，在匹配的返回点观测返回值。标准 Windows x64 的前四个整数/指针参数通常对应 RCX、RDX、R8、R9；浮点参数对应相应 XMM 寄存器，函数中部不能继续假定入口栈布局。具体日志文本和条件设置只从 [reverse-tooling.md](reverse-tooling.md#条件断点与日志) 读取。
3. 先收集原程序的输入、分支、关键值和输出，再做一个能检验假设的最小 patch。保留原字节/寄存器值，并记录补丁范围；补丁改变了输出，只能证明该执行条件下的影响，还要用原始语义或独立复算核对所恢复的算法。
4. 验证结束后恢复本次修改，重新运行相同输入确认基线；存在多线程或重入时，按线程和调用实例关联入口与返回记录，不将两次不同调用拼成一条证据。

如果需要导出解密后的映像，还要检查入口点、节布局、导入和重定位等恢复条件；一次内存 dump 或工具给出的 OEP 候选不能代替恢复验证，相关路线见 [loader-vm-image-and-kernel-patterns.md](loader-vm-image-and-kernel-patterns.md)。反调试 API、时间差及调试填充值只能作为定位信号，应结合返回值、位掩码和后续分支确认作用；不能从固定常量推断完整算法或调试状态。

### 其他使用原则

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
- [Microsoft x64 调用约定](https://learn.microsoft.com/en-us/cpp/build/x64-calling-convention)
- [x64dbg 条件断点](https://help.x64dbg.com/en/latest/introduction/ConditionalBreakpoint.html)
