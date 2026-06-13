---
type: family
tags: [pwn, family, rop, ret2csu, dynelf, shellcode, badchars]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/ret2csu-dynelf-and-shellcode.md
updated: 2026-06-12
---

# ret2csu, DynELF and Shellcode

## 作用边界

本页是 Pwn 后期控制流落地 family，覆盖 ret2libc、raw syscall ROP、ret2csu、DynELF、bad-character ROP、exotic gadgets、受限 shellcode、小缓冲 stager 和替代 syscall。它负责回答：已经有 RIP/ROP/shellcode 入口后，如何在约束下调用函数、解析符号、放置字符串或执行系统调用。

如果首要问题仍是怎么拿控制流，先看 [overflow-basics.md](overflow-basics.md)、[format-string.md](format-string.md) 或 heap 页面。如果主要约束是 seccomp 或栈迁移，先看 [stack-pivots-srop-and-seccomp-rop.md](stack-pivots-srop-and-seccomp-rop.md)。

## 共同识别信号

- 已能控制返回地址或函数指针，但缺少 `pop rdi/rsi/rdx`、libc 基址、可用字符串或足够 shellcode 空间。
- 程序可泄露任意地址或 GOT 内容，需要解析 libc 符号。
- payload 受 badchars、唯一字节、小缓冲、预初始化寄存器、静态链接或 syscall 限制影响。
- 目标可能是 `system("/bin/sh")`、`execve`、ORW、mprotect+shellcode、DynELF resolve 或短 stager。

## 最小证据

- 可控寄存器集合、可写内存、可泄露地址、可执行内存和 gadget 列表。
- RELRO、PIE、NX、canary、libc 版本和 seccomp 约束。
- 字符串放置方案：BSS、heap、栈、ROP 写入、XOR 解码或已有字符串。
- 最终交互方式：shell、ORW、直接打印 flag、反连或文件读。

## 首轮路由

| 证据形态 | 首轮判断 | 下一跳 |
|---|---|---|
| 能泄露 libc/GOT，能二次输入 | 先做两阶段 ret2libc，泄露后返回干净调用点再发第二链 | [pwn-tooling.md](pwn-tooling.md) |
| 缺少常规参数 gadget，但有 `__libc_csu_init` | 用 ret2csu 设置 `rdi/rsi/rdx` 并调用 GOT/PLT 函数 | [stack-pivots-srop-and-seccomp-rop.md](stack-pivots-srop-and-seccomp-rop.md) |
| 有任意读但不知道 libc | 用 DynELF 或手工解析 ELF link map/GOT/符号表 | [format-string.md](format-string.md) |
| `system()` 不可用或 seccomp 限制 | 转 raw syscall ROP、ORW 或替代 syscall | [seccomp-ret2dlresolve-and-runtime-primitives.md](seccomp-ret2dlresolve-and-runtime-primitives.md) |
| badchars/唯一字节/小缓冲 | 先写编码器或 stager，利用 XOR、sprintf、BEXTR/XLAT/STOSB/PEXT 等 gadget 组装目标字节 | [windows-arm-and-cross-platform-exploits.md](windows-arm-and-cross-platform-exploits.md) |
| shellcode 空间小但寄存器已预设 | 只补缺失参数或第一阶段 read/mprotect，再加载完整 shellcode | [stack-pivots-srop-and-seccomp-rop.md](stack-pivots-srop-and-seccomp-rop.md) |
| shell 出来但命令被 read 吃掉或无回显 | 先处理交互时序、PTY、延迟和 fd，再考虑 ORW 替代 | [pwn-tooling.md](pwn-tooling.md) |

## 合并与拆分结论

本页应为 family。ret2csu、DynELF、badchar ROP、shellcode stager 和 syscall ROP 是不同技术，但它们都发生在“控制流已经可用，如何落地最终能力”的阶段。保留为 family 比拆成短页更利于 exploit 链路阅读。

## 常见陷阱

- 泄露后返回 `main` 破坏栈状态，没返回到干净的 `call vuln` 或初始化点。
- ret2csu 忘记满足 `rbx/rbp` 关系，循环多执行或不执行。
- DynELF 的 leak 函数不稳定，读到 unmapped 地址直接断连。
- shellcode 没处理 badchars、NX 或 seccomp，能本地执行但远程失败。
- 拿到 shell 后发送命令太早，被前面的 `read()` 消耗。

## 关联技巧

- [stack-pivots-srop-and-seccomp-rop.md](stack-pivots-srop-and-seccomp-rop.md)
- [seccomp-ret2dlresolve-and-runtime-primitives.md](seccomp-ret2dlresolve-and-runtime-primitives.md)
- [format-string.md](format-string.md)
- [overflow-basics.md](overflow-basics.md)
- [windows-arm-and-cross-platform-exploits.md](windows-arm-and-cross-platform-exploits.md)
- [pwn-tooling.md](pwn-tooling.md)

## 原始资料

- [ret2csu-dynelf-and-shellcode.md](../raw/pwn/ret2csu-dynelf-and-shellcode.md)
