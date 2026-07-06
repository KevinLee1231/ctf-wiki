---
type: family
tags: [reverse, family, signal-handler, trace, packed, dump]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/signal-trace-and-packed-anti-analysis.md
updated: 2026-06-12
---

# Signal, Trace and Packed Anti-Analysis

## 作用边界

本页是信号处理器、trace 反演、父子进程 patch、packed/dynamic module dump 和异常控制流 family。它适合处理 SIGILL/SIGFPE、`strace`/`ptrace` 可观测副作用、指令 trace、call-less chaining、ConfuserEx 动态模块和父进程修改子进程这类运行时结构。

它不替代 [anti-analysis.md](anti-analysis.md)：反分析页负责跨过环境检测，本页负责利用 signal/trace/dump 证据恢复真实逻辑。

## 识别信号

- 程序注册 SIGILL/SIGFPE/SIGSEGV handler，异常不是 crash，而是控制流或 side-channel。
- `strace`、coredump、instruction trace、process_vm_writev/readv 能暴露父子进程或 packed stage。
- 静态二进制不是最终执行体，真实模块在运行时生成、解密、写入子进程或动态加载。
- 函数调用被栈帧、异常、signal handler 或 call-less chaining 隐藏。

## 最小证据

- 信号类型、handler 地址、触发条件和 handler 修改的寄存器/内存。
- Trace 采集方式和粒度：syscall、basic block、instruction、coredump、module constructor。
- packed/dynamic module 的 dump 时机：解密后、执行前、构造函数断点、父进程 patch 后。
- 恢复结果能映射回真实校验、key、opcode 或明文，而不是只得到一段不可定位 dump。

## 路由表

| 证据 | 先验证 | 下一跳 |
|---|---|---|
| SIGILL mode switching | handler 是否修改 PC/flags/寄存器实现隐藏指令语义 | 建 handler-aware trace |
| SIGFPE side-channel | 信号次数或触发位置是否和输入字符相关 | `strace` 计数或自动化 oracle |
| instruction trace inversion | trace 能否还原执行序列，指令是否可用 Keystone/Unicorn 重放 | 转 [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md) |
| call-less chaining | 栈帧被手工构造，普通 xref 缺失 | 先恢复栈布局和返回地址链 |
| parent-patched child | 父进程写入子进程或 memfd 后再执行 | dump 子进程修改后镜像 |
| ConfuserEx dynamic module | .NET 模块在 constructor 或 runtime 动态解密 | constructor 断点 dump module |
| packed anti-analysis | 壳和检测交织，trace 指向 fake path | 先转 [anti-analysis.md](anti-analysis.md) 清环境 |

## 合并与拆分结论

- 保留为 family：信号、trace、dump、父子进程和 packed module 是一组运行时证据路线。
- 不合并进 `runtime-patching-oracles-and-tracing.md`：本页的触发证据更偏 signal/trace/dump 结构，能提供更具体 pivot。
- 不合并进 `packers-deobfuscation-and-debug-automation.md`：该页偏壳/脱壳自动化，本页偏异常和 trace 证据。

## 常见误判

- 把信号当崩溃处理，直接 patch 掉 handler，丢失真实控制流。
- Dump 时机太早，模块仍是加密或占位状态。
- Trace 只记录 syscall，实际分支在用户态指令层。
- 父子进程题只分析初始文件，没有检查运行后 child memory。

## 关联页面

- [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md)
- [anti-analysis.md](anti-analysis.md)
- [packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md)
- [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md)
- [qiling-triton-pin-and-ldpreload.md](qiling-triton-pin-and-ldpreload.md)
- [reverse-tooling.md](reverse-tooling.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [HGAME2026-signal-storm-wp](../raw/reverse/HGAME2026-signal-storm-wp.md) | SIGSEGV/SIGTRAP/SIGFPE handler 改 RC4 状态，`TracerPid` 混入 key；先 patch 反调试或复现 handler 后断 `memcmp`。 |
| [RCTF2025-chaos2-wp](../raw/reverse/RCTF2025-chaos2-wp.md) | 大量花指令、反调试和动态 key 修改掩护 RC4 解密，先清理 junk、跟构造函数和密钥更新点。 |

## 原始资料

- [signal-trace-and-packed-anti-analysis.md](../raw/reverse/signal-trace-and-packed-anti-analysis.md)
