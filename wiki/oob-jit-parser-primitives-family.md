---
type: family
tags: [pwn, family]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/oob-jit-parser-primitives.md
updated: 2026-05-22
---

# OOB / JIT / Parser 原语技巧族

## 适用场景

本页是 `OOB / JIT / Parser 原语技巧族` 技巧家族页，用来承接多个相邻技巧和案例；先用于判断是否属于这一族，再选择具体变体。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 稳定 crash、ASAN 报告、异常退出、非法地址访问或远程连接中断。
- 漏洞面是解释器、JIT、parser、游戏关卡、文件格式、emulator 或自定义 VM，而不是普通菜单堆题。
- 读写能力不完整：只能 OOB read、partial write、index truncation、signed/unsigned mismatch、FD inheritance 或只读 oracle。
- 保护组合会决定路线：Full RELRO、PIE、Canary、NX、seccomp、sandbox、JIT W^X、custom allocator。

## 最小证据

- 有最小输入能稳定触发 bug，并记录 crash 地址、寄存器、栈/堆对象和偏移。
- 已量化 primitive：能读哪里、写哪里、写多少、是否可重复、是否受字符集或解析器限制。
- 已确认目标环境保护和 libc/loader 差异；远程 exploit 不应只依赖本地偏移。
- 能说明从 primitive 到目标能力的链路：leak、任意读写、控制流、shell、文件读取或 sandbox escape。

## 解法骨架

1. 最小化触发样本，固定输入长度、字段、序列化格式和交互时序。
2. 用调试器或 sanitizer 量化 primitive，不急于拼 ROP。
3. 先解决信息泄漏，再决定 GOT/HOOK/ret2libc/ROP/SROP/ret2dlresolve/JIT spray/sandbox escape。
4. 把 exploit 拆成阶段：握手、leak、基址计算、写入或劫持、触发、交互。
5. 对远程差异设置超时、recv 边界、重试和日志，保留一次成功 transcript。

## 关键变体

| 变体 | 优先证据 | 下一跳页面 | 失败后 pivot |
|---|---|---|---|
| OOB read/write | index、length、count 或坐标越界 | [interpreter-jit-canary-and-integer-exploits.md](interpreter-jit-canary-and-integer-exploits.md) | 若只能读不能写，转 leak + 逻辑绕过或只读 ROP 构造。 |
| Parser 整数/符号错误 | signed/unsigned、截断、size_t underflow | [emulator-float-and-hash-exploits.md](emulator-float-and-hash-exploits.md) | 若 crash 不可控，先做格式 grammar fuzz 和字段二分。 |
| JIT / emulator escape | guest 指令影响 host 指针或跳转 | [cross-primitive-escape-and-hybrid-exploit-map.md](cross-primitive-escape-and-hybrid-exploit-map.md) | 若 W^X 阻断 shellcode，转 ROP、JOP 或已有 native helper。 |
| GOT/PLT 覆写 | Partial RELRO、任意写、可触发函数调用 | [ret2csu-dynelf-and-shellcode.md](ret2csu-dynelf-and-shellcode.md) | Full RELRO 时转 libc hook、栈迁移、vtable、FSOP 或 ret2dlresolve。 |
| seccomp/sandbox | syscall 被过滤，能读写内存但不能直接 execve | [seccomp-ret2dlresolve-and-runtime-primitives.md](seccomp-ret2dlresolve-and-runtime-primitives.md) | 转 open/read/write、ORW、SROP、ret2dlresolve 或文件描述符继承。 |
| kernel/user boundary | ioctl、copy_from_user、slab、race | [linux-kernel-exploit-basics.md](linux-kernel-exploit-basics.md) | 若目标是理解状态机而非提权，先 pivot 到 reverse 页面。 |

## 常见陷阱

- 只追 crash，不量化 primitive；没有 primitive 表就很难稳定迁移到远程。
- 看到 OOB 就默认任意写；parser bug 经常只有只读、一次性或受字符集限制。
- 忽略远程 libc、ld、ASLR seed、timeout 和交互 flush，导致本地脚本不可复现。
- 在 seccomp 下继续尝试 `system('/bin/sh')`；应先列允许 syscall。
- 对 JIT 题过早写 shellcode；先确认 JIT code page 权限和 guest-to-host 映射。


## 关联技巧

- [cross-primitive-escape-and-hybrid-exploit-map.md](cross-primitive-escape-and-hybrid-exploit-map.md)
- [emulator-float-and-hash-exploits.md](emulator-float-and-hash-exploits.md)
- [format-string.md](format-string.md)
- [heap-fsop-file-structure-attacks.md](heap-fsop-file-structure-attacks.md)
- [heap-houses-unlink-and-tcache.md](heap-houses-unlink-and-tcache.md)
- [pwn-first-pass-red-flags-and-protections.md](pwn-first-pass-red-flags-and-protections.md)
- [overflow-basics.md](overflow-basics.md)
- [python-vm-and-proc-sandbox-escape.md](python-vm-and-proc-sandbox-escape.md)
- [race-condition-and-concurrency-exploits.md](race-condition-and-concurrency-exploits.md)
- [runtime-protection-and-tls-exploits.md](runtime-protection-and-tls-exploits.md)
- [stack-pivots-srop-and-seccomp-rop.md](stack-pivots-srop-and-seccomp-rop.md)
- [windows-arm-and-cross-platform-exploits.md](windows-arm-and-cross-platform-exploits.md)
- [kaslr-kpti-smep-and-kernel-debugging.md](kaslr-kpti-smep-and-kernel-debugging.md)
- [kernel-uaf-race-and-slab-techniques.md](kernel-uaf-race-and-slab-techniques.md)

## 原始资料

- [oob-jit-parser-primitives.md](../raw/pwn/oob-jit-parser-primitives.md)
