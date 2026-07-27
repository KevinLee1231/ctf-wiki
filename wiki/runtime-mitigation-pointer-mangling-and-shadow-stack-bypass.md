---
type: technique
tags: [pwn, mitigation, pointer-mangling, shadow-stack, cet]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/runtime-protection-and-tls-exploits.md
  - ../raw/pwn/windows-arm-and-cross-platform-exploits.md
updated: 2026-07-27
---

# Runtime Mitigation, Pointer Mangling and Shadow-Stack Bypass

## 适用场景

控制流目标受 pointer guard、safe-linking、CET/shadow stack、CFG 或运行时完整性保护约束，必须先恢复编码密钥、选择合法间接调用或转向数据流/清理处理器。

## 识别信号

- 函数/退出指针以 XOR+rotate、heap-base XOR 等形式存储。
- 返回地址覆盖后触发 shadow-stack/CET/CFG 检查。
- 普通 hook 已移除，且可写目标受 vtable/range 校验。

## 最小证据

- 固定目标 ABI、libc/loader/Windows build 与保护配置。
- 从反汇编还原精确 encode/decode 公式和 guard 来源。
- 证明选定目标通过 CFG/vtable/间接调用检查。

## 解法骨架

1. 识别保护作用于返回、间接调用还是数据指针。
2. 从已知明文-密文指针对或 TLS/heap 泄露恢复 guard/base。
3. 若返回链被保护，转向合法 call target、FILE/exit chain 或数据-only primitive。
4. 编码目标指针并在真实版本上验证触发路径。

## 关键变体

- glibc pointer guard / safe-linking。
- Intel CET shadow stack / IBT。
- Windows CFG/SEH 与平台特有调用目标验证。

## 常见陷阱

- 使用错误 rotate 位数、端序或 chunk 地址编码。
- 只绕过一层保护，下一层间接调用仍被拦截。
- 本地未启用与远端相同的硬件/内核保护。

## 关联技巧

- [runtime-protection-and-tls-exploits.md](runtime-protection-and-tls-exploits.md)
- [windows-arm-and-cross-platform-exploits.md](windows-arm-and-cross-platform-exploits.md)
- [file-structure-and-exit-handler-control-flow.md](file-structure-and-exit-handler-control-flow.md)

## 原始资料

- [runtime-protection-and-tls-exploits.md](../raw/pwn/runtime-protection-and-tls-exploits.md)
- [windows-arm-and-cross-platform-exploits.md](../raw/pwn/windows-arm-and-cross-platform-exploits.md)
