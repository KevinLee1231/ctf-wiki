---
type: technique
tags: [reverse, anti-debug, anti-vm, self-check, patching]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/anti-analysis.md
  - ../raw/reverse/signal-trace-and-packed-anti-analysis.md
  - ../raw/reverse/WMCTF2020-welcome-to-ctf-wp.md
updated: 2026-07-28
---

# Anti-Debug, Self-Check and Environment Bypass

## 适用场景

程序通过 debugger/VM/DBI 检测、timing、异常/信号、自校验或环境探针改变控制流，需要定位检测结果的消费点并最小化 patch/伪装。

## 识别信号

- `ptrace`、PEB flags、异常处理、时间差、CPUID、设备/进程检查。
- 调试时退出/假路径，正常运行却继续。
- 代码页 hash、CRC 或自修改逻辑会验证 patch。

## 最小证据

- 对比有无调试器/虚拟化时的首个控制流差异。
- 找到检测值如何影响 branch、key 或解密状态。
- Patch 后主要算法输出保持一致，不只是跳过退出。

## 解法骨架

1. 在检测 API、异常和 branch 消费点设置日志/断点。
2. 优先 hook 返回值或修改单个判定，避免大段 NOP。
3. 自校验时改输入/期望值或在校验完成后动态 patch。
4. 用未调试运行和独立输入复验真实路径。

## 关键变体

- API/flag anti-debug。
- Timing/exception/signal anti-analysis。
- Self-checksum、anti-DBI 与自修改代码。

## 常见陷阱

- 跳过检测同时跳过初始化/解密。
- Patch 触发后续自校验但未察觉。
- 将普通崩溃误判为 anti-debug。

## 关联技巧

- [anti-analysis.md](anti-analysis.md)
- [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md)
- [trace-hook-and-state-snapshot-reconstruction.md](trace-hook-and-state-snapshot-reconstruction.md)
- [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md)

## 原始资料

- [anti-analysis.md](../raw/reverse/anti-analysis.md)
- [signal-trace-and-packed-anti-analysis.md](../raw/reverse/signal-trace-and-packed-anti-analysis.md)
- [WMCTF2020-welcome-to-ctf-wp](../raw/reverse/WMCTF2020-welcome-to-ctf-wp.md)
