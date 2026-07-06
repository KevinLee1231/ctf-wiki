---
type: family
tags: [pwn, family, seccomp, ret2dlresolve, syscall, runtime]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/seccomp-ret2dlresolve-and-runtime-primitives.md
updated: 2026-06-12
---

# Seccomp, ret2dlresolve and Runtime Primitives

## 作用边界

本页是受限 syscall 与动态解析 family，用于在已经有用户态控制流或读写 primitive 后，判断如何把它转成文件读取、shell、动态符号解析或运行时可调用原语。它不是通用 heap/UAF 页，也不是所有 seccomp 绕过技巧的 payload 列表。

如果主要问题是先拿控制流，回到 [overflow-basics.md](overflow-basics.md)、[format-string.md](format-string.md) 或 heap 页面；如果主要问题是换栈/SROP/ROP gadget 组织，转 [stack-pivots-srop-and-seccomp-rop.md](stack-pivots-srop-and-seccomp-rop.md)。

## 识别信号

- `seccomp-tools dump` 或反汇编显示 syscall allowlist/denylist，`execve`、`open`、`read`、`write` 或 `mprotect` 受限。
- 已有可控 RIP、ROP、shellcode、任意读写、GOT 写、BSS 写或格式串写，但最终能力受运行时限制。
- 二进制缺 libc leak、GOT/PLT 可用、Full RELRO/No RELRO 差异影响 ret2dlresolve 可行性。
- raw 线索涉及 `openat/openat2`、ORW、ret2dlresolve、GOT overwrite、JIT/runtime syscall stub、rdx/rax 控制或运行时短原语。

## 最小证据

- 已明确 seccomp 过滤条件：允许哪些 syscall、是否检查参数、是否有 x32 ABI/openat2/`open_how` 等差异。
- 已定义当前 primitive：能写哪里、能读哪里、能控哪些寄存器、可用 writable 段和可调用函数。
- 对 ret2dlresolve，确认 PLT0、GOT、link_map、dynsym/dynstr/rela 地址和可写伪造结构位置。
- 对 shellcode/ORW，确认 syscall ABI、fd 来源、栈/寄存器约束和输入编码限制。

## 首轮路由

| 证据形态 | 先做什么 | 下一跳 |
|---|---|---|
| seccomp 只允许 ORW 或禁 `execve` | dump filter，选择 `open/openat/openat2 + read/mmap + write/sendfile`，动态捕获 fd 不要假设是 3。 | [stack-pivots-srop-and-seccomp-rop.md](stack-pivots-srop-and-seccomp-rop.md) |
| filter 检查 syscall number 但存在 ABI/参数差异 | 检查 x32 ABI、`openat2` 参数结构、BPF X-register、arch switch 或 raw BPF 解析差异。 | [stack-pivots-srop-and-seccomp-rop.md](stack-pivots-srop-and-seccomp-rop.md) |
| 无 libc leak 但可写 BSS 且 PLT 可用 | 构造 fake `Elf*_Rela/Sym/dynstr`，必要时处理 VERSYM/link_map，走 ret2dlresolve。 | [ret2csu-dynelf-and-shellcode.md](ret2csu-dynelf-and-shellcode.md) |
| 可写 GOT/函数指针/`.fini_array` | 先选触发稳定且不破坏后续输出的目标；若是格式串写，转格式串页细化写入粒度。 | [format-string.md](format-string.md) |
| shellcode 可执行但编码/重定位受限 | 构造无 relocation、call/pop 定位、短 stage 或输入变换可逆的 shellcode。 | [windows-arm-and-cross-platform-exploits.md](windows-arm-and-cross-platform-exploits.md) |
| 运行时/JIT/解释器提供 syscall stub | 先验证 stub 可达和寄存器残留，再把 read 返回值、rdx/r10 等副作用纳入 ROP。 | [interpreter-jit-canary-and-integer-exploits.md](interpreter-jit-canary-and-integer-exploits.md) |

## 合并与拆分结论

- 保留为 `family`：raw 覆盖 seccomp、openat2、参数过滤、ret2dlresolve、GOT/`.fini_array`、runtime syscall stub 和短案例运行时 primitive。
- 不并入 [stack-pivots-srop-and-seccomp-rop.md](stack-pivots-srop-and-seccomp-rop.md)：本页选择“最终能力路线”，stack 页组织“如何把执行流和寄存器摆到位”。
- 不拆 ret2dlresolve 独立页：当前已有 [ret2csu-dynelf-and-shellcode.md](ret2csu-dynelf-and-shellcode.md) 承担动态解析与加载器相关技术细节，本页保留为路由入口。

## 常见陷阱

- 只看 syscall allowlist，不看参数检查、arch 字段和 BPF 原始指令。
- ORW 链假设 fd=3，Docker/远程环境中常因已有 fd 变成 4/5。
- ret2dlresolve 只伪造 Rela/Sym，忽略 64-bit VERSYM 检查或 link_map 偏移版本差异。
- seccomp 题仍追求 `/bin/sh`，没有先判断读 flag 文件是否更短。
- shellcode 用绝对地址或 relocation，远程 ASLR/PIE 下直接失效。

## 关联技巧

- [pwn-first-pass-red-flags-and-protections.md](pwn-first-pass-red-flags-and-protections.md)
- [stack-pivots-srop-and-seccomp-rop.md](stack-pivots-srop-and-seccomp-rop.md)
- [ret2csu-dynelf-and-shellcode.md](ret2csu-dynelf-and-shellcode.md)
- [format-string.md](format-string.md)
- [interpreter-jit-canary-and-integer-exploits.md](interpreter-jit-canary-and-integer-exploits.md)
- [runtime-protection-and-tls-exploits.md](runtime-protection-and-tls-exploits.md)
- [pwn-tooling.md](pwn-tooling.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [LilacCTF2026-bytezoo-wp](../raw/pwn/LilacCTF2026-bytezoo-wp.md) | 受限执行环境需要构造运行时读写/执行 primitive，先验证 mprotect、memmove 和二阶段 shellcode。 |
| [RCTF2025-only-rev-wp](../raw/pwn/RCTF2025-only-rev-wp.md) | syscall stub 或运行时寄存器残留提供读写/执行入口，先验证可控寄存器、可写段和 shellcode 通道。 |
| [RCTF2025-only-wp](../raw/pwn/RCTF2025-only-wp.md) | 浮点/机器码输入进入 syscall stub，先确认可执行内存、read primitive 和二阶段 shellcode。 |
| [SU_evbufferWP](../raw/pwn/SU_evbufferWP.md) | TCP/UDP `inet_pton` 后全局溢出伪造 libevent `bufferevent/evbuffer` callback，seccomp 下栈迁移到 ORW。 |
| [VNCTF2026-vm-syscall-wp](../raw/pwn/VNCTF2026-vm-syscall-wp.md) | VM `case4` 把 4 个寄存器映射到 syscall，但执行前清零 `r8/r9/r10`；绕过常规 `mmap`，用 `shmget/shmat` 放 `/bin/sh` 后 `execve`。 |

## 原始资料

- [seccomp-ret2dlresolve-and-runtime-primitives.md](../raw/pwn/seccomp-ret2dlresolve-and-runtime-primitives.md)
