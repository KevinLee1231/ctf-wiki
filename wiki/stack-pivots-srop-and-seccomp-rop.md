---
type: family
tags: [pwn, family, rop, srop, stack-pivot, seccomp]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/stack-pivots-srop-and-seccomp-rop.md
updated: 2026-06-12
---

# Stack Pivots, SROP and Seccomp ROP

## 作用边界

本页是 Pwn 控制流构造 family 页，覆盖栈迁移、SROP、受编码限制的 payload、RETF 架构切换、vDSO/vsyscall、JIT-ROP、ret2dlresolve、受限 gadget ROP、`.fini_array` 分阶段劫持和静态链接 ret2libc 等路线。

这些内容的共同点是：已经有某种控制流入口，但入口空间、寄存器、syscall、栈位置或字符集受到限制。首轮任务不是马上堆 ROP，而是判断“控制流入口能承载哪一种执行模型”。

## 共同识别信号

- 能控制返回地址、函数指针、`.fini_array`、异常返回帧、vtable 或某个间接调用，但栈空间不足或 payload 受限。
- 出现 `leave; ret`、`sigreturn`、`syscall`、`int 0x80`、`retf`、vDSO/vsyscall、SROP frame、BSS stack、UTF-8/反转输入等线索。
- seccomp 允许的 syscall 与预期 shell 路线冲突，必须改成 ORW、mmap+write、x32 ABI、JIT-ROP 或时间侧信道。
- 本地可打通但远程失败时，常见原因是栈 pivot 地址、信号帧、libc 版本、vDSO 映射或 fd 差异。

## 最小证据

- 控制流入口的具体位置和可用字节数。
- 可控寄存器集合，尤其是 `rsp/rbp/rdi/rsi/rdx/rax` 或架构对应参数寄存器。
- 可写可读内存区域，如 BSS、heap、栈残留、mmap 区、JIT 区。
- syscall 可用性和 seccomp 过滤结果。
- payload 编码、长度、反转、空字节、UTF-8 或唯一字节限制。

## 首轮路由

| 证据形态 | 首轮判断 | 下一跳 |
|---|---|---|
| 栈空间很小但能控 `rbp/rsp` 或有 `leave; ret` | 先做一段或两段栈迁移，把第二阶段放到 BSS/heap/mmap | [ret2csu-dynelf-and-shellcode.md](ret2csu-dynelf-and-shellcode.md) |
| 能控制 `rax=15` 或触发 `rt_sigreturn` | 构造 SROP frame 前先确认 frame 地址、`syscall` gadget 和 seccomp 限制 | [seccomp-ret2dlresolve-and-runtime-primitives.md](seccomp-ret2dlresolve-and-runtime-primitives.md) |
| seccomp 禁止常规 execve/read/write 组合 | 先枚举允许 syscall，再选 ORW、x32 ABI、RETF、vDSO、JIT-ROP 或 blind shellcode | [seccomp-ret2dlresolve-and-runtime-primitives.md](seccomp-ret2dlresolve-and-runtime-primitives.md), [pwn-tooling.md](pwn-tooling.md) |
| 只有格式化写或有限任意写 | 优先考虑 `.fini_array`、GOT、exit handler 或分阶段 ROP，而不是一次写完完整链 | [format-string.md](format-string.md), [runtime-protection-and-tls-exploits.md](runtime-protection-and-tls-exploits.md) |
| 可泄露 libc 函数但 gadget 不稳定 | 先从泄露范围内扫描 syscall、one_gadget 或可拼接序列 | [ret2csu-dynelf-and-shellcode.md](ret2csu-dynelf-and-shellcode.md) |
| gadget 或立即数受素数、字符集、唯一字节等约束 | 先把约束建模，再决定分解常量、分阶段写入或切换 shellcode 模型 | [pwn-tooling.md](pwn-tooling.md), [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md) |

## 合并与拆分结论

本页保留为 family。SROP、栈迁移、RETF、vDSO、ret2dlresolve 和 `.fini_array` hijack 都有独立技术细节，但在健康检查层面，它们常被同一批 raw 混在“受限控制流如何落地”问题下。将其改成 family 可以减少误导：具体执行骨架落到相邻 technique，首轮 pivot 留在这里。

## 常见陷阱

- 只验证了本地 gadget，没有检查远程 vDSO、libc、栈地址和 seccomp。
- SROP frame 地址和 `rsp` 落点不一致，导致看似 syscall 失败。
- `leave; ret` 栈迁移只做一次，但第二阶段仍然覆盖不完整。
- ret2dlresolve 与 seccomp 同时出现时，只完成解析却调用了被过滤的 syscall。
- `.fini_array` 分阶段劫持忘记安排第二次触发，导致只执行一次短链。

## 关联技巧

- [seccomp-ret2dlresolve-and-runtime-primitives.md](seccomp-ret2dlresolve-and-runtime-primitives.md)
- [ret2csu-dynelf-and-shellcode.md](ret2csu-dynelf-and-shellcode.md)
- [format-string.md](format-string.md)
- [runtime-protection-and-tls-exploits.md](runtime-protection-and-tls-exploits.md)
- [interpreter-jit-canary-and-integer-exploits.md](interpreter-jit-canary-and-integer-exploits.md)
- [pwn-tooling.md](pwn-tooling.md)

## 原始资料

- [stack-pivots-srop-and-seccomp-rop.md](../raw/pwn/stack-pivots-srop-and-seccomp-rop.md)
