---
type: tooling
tags: [reverse, tooling, disassembler, debugger]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/disassemblers-debuggers-and-basic-tools.md
updated: 2026-07-06
---

# Disassemblers, Debuggers and Basic Tools

## 作用边界

这是逆向题的第一层工具路由页：用于把未知 ELF/PE/APK/WASM/pyc/.NET/固件先拆成可观察的入口、字符串、导入、交叉引用、运行时状态和最小可复现行为。它不负责记录某一种攻击技巧，而是说明静态反汇编、动态调试、轻量模拟和格式专用工具分别该在什么时候上场。

当问题已经明确落到反调试、壳/虚拟化、自定义 VM、符号执行或运行时 hook 时，应从这里转入相应 technique 或更专门的 tooling 页。

## 触发证据

- 只知道附件是 ELF/PE/APK/WASM/pyc/.NET/固件或未知二进制，还没有确认入口、架构、保护、字符串、导入和运行行为。
- 需要用最小命令建立事实：`file`、`strings`、`readelf`、`objdump`、`checksec`、GDB 断 `main` 或 Ghidra/r2 交叉引用。
- raw 证据只证明“可运行/可反汇编/能看到比较点”，还没出现反调试、壳、VM、完整模拟或符号执行的必要性。
- 需要判断后续应该转 [anti-analysis.md](anti-analysis.md)、[compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md)、[frida-angr-lldb-and-x64dbg.md](frida-angr-lldb-and-x64dbg.md) 还是 [qiling-triton-pin-and-ldpreload.md](qiling-triton-pin-and-ldpreload.md)。

## 工具路由

| 任务 | 优先工具 | 使用边界 |
|---|---|---|
| 快速确认入口、字符串、导入、节区、PIE 基址 | `file` / `strings` / `readelf` / `objdump` / `checksec` / GDB | 先建立载体事实，不要在还没确认架构和保护时进入复杂反编译。 |
| 运行时断点、寄存器、内存、比较点和 PIE 调试 | GDB + pwndbg | 适合能本地运行的 ELF；先断 `main`、`memcmp`、`strcmp`、解密函数或错误分支，再决定是否转 trace 或 patch。 |
| CLI 反汇编、交叉引用、脚本化抽取 | Radare2 / r2pipe | 适合批量定位函数、常量和 basic block；反编译质量不足时不要强行依赖伪代码。 |
| 结构恢复、交叉引用、伪代码和类型推断 | Ghidra / headless Ghidra | 适合算法恢复和大程序导航；遇到 SMC、packed code 或运行时生成逻辑时需要配合动态 dump。 |
| 小段代码解密、指令级模拟、混合模式片段 | Unicorn | 适合从二进制中切出稳定函数或 shellcode；如果依赖完整系统调用、文件系统或动态库，转 Qiling。 |
| Python 字节码恢复 | `pycdc` / `pycdas` / `uncompyle6` | 先确认 Python 版本和 magic；反编译失败时退回字节码和常量表，而不是只换反编译器。 |
| WASM 分析 | `wasm2wat` / `wasm-decompile` / 浏览器调试器 | 先看导入导出、线性内存和校验函数；复杂控制流再转 trace 或手写解释。 |
| Android APK | `apktool` / `jadx` / `baksmali` | 先分 Java/Kotlin 层和 native 层；JNI 或 native 校验应转入普通二进制逆向。 |
| .NET 样本 | dnSpy / ILSpy | 先看 IL、资源和混淆器痕迹；动态加载或加密资源需要运行时 dump。 |
| UPX、PyInstaller、简单 packer | `upx` / `pyinstxtractor` / 运行时 dump | 工具能解就解，不能解就从 OEP、解密后内存和最终比较点入手。 |

## 最小工作流

1. 先记录格式、架构、保护、入口、导入、字符串和可运行性。
2. 用静态工具找可能的校验点；用动态调试确认输入如何影响分支。
3. 若能稳定到达比较点，优先抽取约束或明文，而不是完整理解所有外围逻辑。
4. 只有在静态/动态基础工具无法跨过反调试、壳、VM 或外部环境时，才升级到 Frida、Qiling、Triton、Pin、LD_PRELOAD 或专门 technique。

## 失败信号与转向

- 伪代码有大段错误类型、不可达分支或动态生成函数：转 [packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md) 或 [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md)。
- 程序一运行就退出、检测调试器、校验时间/环境：转 [anti-analysis.md](anti-analysis.md)。
- 只差最终输入，且比较点可断：转 [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md)。
- 需要 hook 系统 API、替换函数返回值或移动端运行时插桩：转 [frida-angr-lldb-and-x64dbg.md](frida-angr-lldb-and-x64dbg.md)。
- 依赖完整文件系统、系统调用或跨平台模拟：转 [qiling-triton-pin-and-ldpreload.md](qiling-triton-pin-and-ldpreload.md)。

## 关联页面

- [reverse-tooling.md](reverse-tooling.md)
- [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md)
- [anti-analysis.md](anti-analysis.md)
- [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md)
- [embedded-python-pyd-custom-aes.md](embedded-python-pyd-custom-aes.md)

## 原始资料

- [disassemblers-debuggers-and-basic-tools.md](../raw/reverse/disassemblers-debuggers-and-basic-tools.md)
