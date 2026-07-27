---
type: family
tags: [pwn, family, oob, jit, parser, primitive, sandbox, emulator]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/oob-jit-parser-primitives.md
  - ../raw/pwn/D3CTF2019-basic-basic-parser-wp.md
  - ../raw/pwn/D3CTF2021-easy-chrome-full-chain-wp.md
updated: 2026-07-27
---

# OOB / JIT / Parser 原语技巧族

## 作用边界

本页是 Pwn primitive 形成 family，负责把 OOB、JIT、parser、emulator、整数边界和 sandbox 触发面分流到“先形成什么能力、再落到哪个目标”。

它不替代 Pwn 首轮页；只有在已经有越界、解释器/JIT/parser crash、host/guest 语义落差或可疑低级读写能力时才进入本页。

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

## 分流流程

1. 最小化触发样本，固定输入长度、字段、序列化格式和交互时序。
2. 用调试器或 sanitizer 量化 primitive，不急于拼 ROP。
3. 先解决信息泄漏，再决定 GOT/HOOK/ret2libc/ROP/SROP/ret2dlresolve/JIT spray/sandbox escape。
4. 把 exploit 拆成阶段：握手、leak、基址计算、写入或劫持、触发、交互。
5. 对远程差异设置超时、recv 边界、重试和日志，保留一次成功 transcript。

## 路线分流

| 变体 | 优先证据 | 下一跳页面 | 失败后 pivot |
|---|---|---|---|
| OOB read/write | index、length、count 或坐标越界 | [interpreter-jit-canary-and-integer-exploits.md](interpreter-jit-canary-and-integer-exploits.md) | 若只能读不能写，转 leak + 逻辑绕过或只读 ROP 构造。 |
| Parser 整数/符号错误 | signed/unsigned、截断、size_t underflow | [data-interpretation-memory-primitives.md](data-interpretation-memory-primitives.md) | 若 crash 不可控，先做格式 grammar fuzz 和字段二分。 |
| JIT / emulator escape | guest 指令影响 host 指针或跳转 | [cross-primitive-escape-and-hybrid-exploit-map.md](cross-primitive-escape-and-hybrid-exploit-map.md) | 若 W^X 阻断 shellcode，转 ROP、JOP 或已有 native helper。 |
| GOT/PLT 覆写 | Partial RELRO、任意写、可触发函数调用 | [ret2csu-dynelf-and-shellcode.md](ret2csu-dynelf-and-shellcode.md) | Full RELRO 时转 libc hook、栈迁移、vtable、FSOP 或 ret2dlresolve。 |
| seccomp/sandbox | syscall 被过滤，能读写内存但不能直接 execve | [seccomp-ret2dlresolve-and-runtime-primitives.md](seccomp-ret2dlresolve-and-runtime-primitives.md) | 转 open/read/write、ORW、SROP、ret2dlresolve 或文件描述符继承。 |
| kernel/user boundary | ioctl、copy_from_user、slab、race | [linux-kernel-exploit-basics.md](linux-kernel-exploit-basics.md) | 若目标是理解状态机而非提权，先 pivot 到 reverse 页面。 |

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| JIT 类型反馈/边界优化产生 OOB、addrof/fakeobj | [jit-oob-and-runtime-object-corruption.md](jit-oob-and-runtime-object-corruption.md) |
| 协议长度/状态/parser 消费量失配产生破坏 | [protocol-length-state-parser-corruption.md](protocol-length-state-parser-corruption.md) |
| 整数、浮点、指针和原始字节解释差异形成 primitive | [data-interpretation-memory-primitives.md](data-interpretation-memory-primitives.md) |

## 常见陷阱

- 只追 crash，不量化 primitive；没有 primitive 表就很难稳定迁移到远程。
- 看到 OOB 就默认任意写；parser bug 经常只有只读、一次性或受字符集限制。
- 忽略远程 libc、ld、ASLR seed、timeout 和交互 flush，导致本地脚本不可复现。
- 在 seccomp 下继续尝试 `system('/bin/sh')`；应先列允许 syscall。
- 对 JIT 题过早写 shellcode；先确认 JIT code page 权限和 guest-to-host 映射。

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [D3CTF2019-basic-basic-parser-wp](../raw/pwn/D3CTF2019-basic-basic-parser-wp.md) | 源码 parser 的 grammar token fuzz 暴露错误恢复路径 UAF；先用 ASan 定位对象生命周期，再量化 heap primitive 和退出阶段可控调用。 |
| [D3CTF2021-easy-chrome-full-chain-wp](../raw/pwn/D3CTF2021-easy-chrome-full-chain-wp.md) | V8 bounds check 消除后的 OOB 不是终点，还要量化 addrof/fake_obj/ArrayBuffer 任意读写，并把 primitive 接到 Mojo sandbox escape。 |
| [D3CTF2021-real-vmpwn-wp](../raw/pwn/D3CTF2021-real-vmpwn-wp.md) | 自定义 VM/解释器对象漏洞，先定义 VM 指令面到宿主内存 primitive。 |
| [LilacCTF2026-elk-wp](../raw/pwn/LilacCTF2026-elk-wp.md) | 嵌入式 JS 引擎对象/属性和 NaN-boxing 形成调用原语，先定义伪造对象到 C 函数调用的 primitive。 |
| [LilacCTF2026-na1vm-wp](../raw/pwn/LilacCTF2026-na1vm-wp.md) | 自定义 VM 任务队列 OOB 覆盖 `head/tail/size` 后控制寄存器和栈基址；任意写还要恢复 glibc `__exit_funcs` pointer mangling key。 |
| [RCTF2025-no-check-wasm-wp](../raw/pwn/RCTF2025-no-check-wasm-wp.md) | V8/WASM 类型校验缺陷允许 `externref` 与 `i64` 互转，先构造 `addrOf/fakeObj`，再泄露栈和 RWX 页写 shellcode。 |
| [SUCTF2026-BoxWP](../raw/pwn/SUCTF2026-BoxWP.md) | J2V8 内嵌 V8 9.3.345.11，先排除 Java 沙箱逃逸，再用 CVE-2021-38003 `JSON.stringify` hole 做 OOB/addrof/RWX wasm。 |
| [0xGame2025-week2-VM-wp](../raw/pwn/0xGame2025-week2-VM-wp.md) | 本题的利用原语不是泛泛的“VM 可以改 GOT”，而是负 `sp` 造成的定向越界：先用 19 次 `pop` 读出 `puts@GOT` 的低 32 位，在 VM 内构造 `puts-system` 的偏移，再用半宽写回把 GOT 改成 `system`。复现时应从 ELF 和 libc 计算 `stack[-18]` 与 `0x300e0`，避免把原题脚本中的常量误当成普适值。 |
| [UMDCTF2025-literally-1984-wp](../raw/pwn/UMDCTF2025-literally-1984-wp.md) | 链条是“错误常量折叠 → 优化版本专属 OOB → 扩大数组长度 → 伪造 ArrayBuffer backing store → 任意读写 → 劫持 Wasm 跳转表”。应分别验证解释器、基线编译器和优化编译器对值域的认识，并锁定目标 V8 提交与构建参数。 |
| [WMCTF2022-broobwser-wp](../raw/pwn/WMCTF2022-broobwser-wp.md) | 利用 LibJS 对象布局破坏，把 `ArrayBuffer` / `DataView` 的元数据改成可控指针，从而获得任意地址读写，再泄露 libc 并布置 ROP；题目给的是单独 JS 解释器、固定 commit 和补丁，而远端只执行脚本文件时，应优先做引擎对象模型和堆布局分析，不要把服务端包装层当主要攻击面。 |
| [WMCTF2022-ctf-team-simulator-wp](../raw/pwn/WMCTF2022-ctf-team-simulator-wp.md) | 把二进制当作一门小 DSL 解释器来逆向，先恢复 token 与语法，再找特权命令的语义约束；服务端收集一整段脚本后再交给本地程序执行，且二进制中出现大量 flex/yacc 风格 token 时，应优先分析语法动作和命令状态机。 |
| [WMCTF2023-corejs-wp](../raw/pwn/WMCTF2023-corejs-wp.md) | 利用 DFG JIT 未正确 clobberize 的副作用制造 `ArrayWithDouble` 与 `ArrayWithContiguous` 类型混淆；JSC 补丁涉及 `ValueAdd`、DFG、`clobberize`、`jsCast` 或 structure id 检查时，应重点寻找 JIT 假设与运行时对象类型不一致的路径。 |
| [WMCTF2023-jit-wp](../raw/pwn/WMCTF2023-jit-wp.md) | 利用 JIT 校验与实际内存访问语义不一致造成的越界读写；题目让用户提交字节码/JIT 程序，并声称做了内存边界检查时，应对比“校验器接受的访问”和“运行时真正生成的访问”。 |
| [WMCTF2024-babysigin-wp](../raw/pwn/WMCTF2024-babysigin-wp.md) | 题目重点不是传统内存破坏，而是满足 LLVM pass 的静态模式匹配；四层调用链、`0x7890`、`0x6666`、全局 `0x8888` 是关键约束。 |
| [WMCTF2024-blindvm-wp](../raw/pwn/WMCTF2024-blindvm-wp.md) | 本题保留为空白型 raw WP，原因是官方文档和源码仓库均缺少细节；没有把其他 VM 题的机制硬套到本题，避免归档后产生错误知识。 |
| [WMCTF2024-evm-wp](../raw/pwn/WMCTF2024-evm-wp.md) | 漏洞本质：虚拟地址隔离和物理内存访问路径混用；利用目标：普通进程通过 store 修改特权进程代码页。 |


## 关联技巧

- [cross-primitive-escape-and-hybrid-exploit-map.md](cross-primitive-escape-and-hybrid-exploit-map.md)
- [data-interpretation-memory-primitives.md](data-interpretation-memory-primitives.md)
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
- [D3CTF2019-basic-basic-parser-wp](../raw/pwn/D3CTF2019-basic-basic-parser-wp.md)
- [D3CTF2021-easy-chrome-full-chain-wp](../raw/pwn/D3CTF2021-easy-chrome-full-chain-wp.md)
