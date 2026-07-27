---
type: family
tags: [pwn, family, rop, ret2csu, dynelf, shellcode, badchars]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/ret2csu-dynelf-and-shellcode.md
updated: 2026-07-27
---

# ret2csu, DynELF and Shellcode

## 作用边界

本页是 Pwn 后期控制流落地 family，覆盖 ret2libc、raw syscall ROP、ret2csu、DynELF、bad-character ROP、exotic gadgets、受限 shellcode、小缓冲 stager 和替代 syscall。它负责回答：已经有 RIP/ROP/shellcode 入口后，如何在约束下调用函数、解析符号、放置字符串或执行系统调用。

如果首要问题仍是怎么拿控制流，先看 [overflow-basics.md](overflow-basics.md)、[format-string.md](format-string.md) 或 heap 页面。如果主要约束是 seccomp 或栈迁移，先看 [stack-pivots-srop-and-seccomp-rop.md](stack-pivots-srop-and-seccomp-rop.md)。

## 识别信号

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

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| ret2csu/短 gadget 组合受限寄存器调用与栈控制 | [stack-control-flow-and-constrained-rop.md](stack-control-flow-and-constrained-rop.md) |
| DynELF/ret2dlresolve 从内存或 resolver 恢复目标符号 | [dynamic-linker-and-symbol-resolution-exploitation.md](dynamic-linker-and-symbol-resolution-exploitation.md) |
| 格式化字符串先提供泄露或任意写 | [format-string.md](format-string.md) |

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

## 来自 WP 的案例索引

本节只保留可复用识别信号，不替代原始题解正文。

| Raw WP | 可复用联系 |
|---|---|
| [0xGame2022-week1-winmts-dream-wp](../raw/pwn/0xGame2022-week1-winmts-dream-wp.md) | 核心方法：用满足字符白名单的短 `read` shellcode 作为第一阶段，再覆盖读入完整 `execve` shellcode，实现分阶段执行；识别特征：固定 RWX 映射、执行用户输入、首阶段长度很短且逐字节白名单校验，但后续 `read` 的数据不再检查。 |
| [0xGame2023-week3-shellcode-but-without-syscall-wp](../raw/pwn/0xGame2023-week3-shellcode-but-without-syscall-wp.md) | 利用链由两个原语组成：任意 8 字节写负责把退出回调指向受控区域，运行时自修改负责绕过静态字节过滤并得到第二次 `read`。针对机器码的黑名单只能检查提交时的字节，无法阻止代码在可写可执行内存中自行构造被禁指令。 |
| [0xGame2025-week2-任意代码执行-wp](../raw/pwn/0xGame2025-week2-任意代码执行-wp.md) | 本题是两阶段 shellcode：先利用早期零字节绕过 `strlen`，再借助入口残留寄存器在 10 字节内构造第二次 `read`，最后用 NOP 滑道解决自覆盖后的续执行位置。原题解中的“10 比特”应为“10 字节”，而 `push 0`、入口寄存器和 `bss+10` 的控制流都是不可省略的关键机制。 |
| [ACTF2023-blind-wp](../raw/pwn/ACTF2023-blind-wp.md) | 本题在无附件场景下通过交互差异恢复了程序的关键栈布局。利用链可以概括为：无边界全局光标造成越界写，改写相邻 `ptr` 后将固定 8 字节输出升级为任意读，再以同一编辑原语完成任意写，最后用 DynELF 解析 `system` 并覆盖返回地址。 |
| [UMDCTF2020-coal-miner-wp](../raw/pwn/UMDCTF2020-coal-miner-wp.md) | 本题的重点是把堆溢出、GOT 改写、libc 泄露和 ret2libc 串成完整链条。启用 canary 并不意味着无法利用：若攻击者能改写 `__stack_chk_fail` 的解析目标，保护机制本身也可能成为可控的跳板。 |
| [UMDCTF2023-pokemon-game-wp](../raw/pwn/UMDCTF2023-pokemon-game-wp.md) | 第一阶段用一字节越界把能力字段改为 `0x07`，同时获得 RWX 权限和必定捕捉能力；第二阶段把随机出现的对象 ID 当作受限字节生成器，逐字节编码 shellcode。 |

## 原始资料

- [ret2csu-dynelf-and-shellcode.md](../raw/pwn/ret2csu-dynelf-and-shellcode.md)
