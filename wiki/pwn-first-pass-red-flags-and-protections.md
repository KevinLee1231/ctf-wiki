---
type: family
tags: [pwn, family, triage]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/pwn-first-pass-red-flags-and-protections.md
updated: 2026-07-06
---

# First-Pass Red Flags and Protections

## 作用边界

刚拿到 pwn 二进制或源码，需要在投入 exploit 前判断漏洞族、保护组合和最短利用路线。这个页不解决某个高级技巧，而是防止首轮方向错：先知道能不能栈溢出、能不能写 GOT、需不需要 leak、是否受 seccomp 限制。

## 识别信号

- 题目给出 ELF、libc、Dockerfile、源码或远程 `nc host port`。
- 源码出现 `gets`、`scanf("%s")`、`strcpy`、`printf(user)`、整数长度、UAF、线程/全局变量。
- `checksec` 保护组合直接影响路线。
- 程序 I/O 简短，但 crash 点、偏移和 libc 版本尚未明确。

## 最小证据

- 已记录 `file`、`checksec`、架构、libc、PIE/RELRO/NX/Canary 状态。
- 至少能复现一次 crash 或异常行为，并知道触发输入长度/格式。
- 对候选漏洞族有最小证据：偏移、格式串泄漏、UAF 行为、整数截断或 race 窗口。

## 分流流程

1. `file ./bin && checksec --file=./bin && ldd ./bin`。
2. 用最短输入触发异常，必要时 `pwn cyclic` / GDB 找偏移。
3. 根据保护选择路线：No PIE 固定地址，Partial RELRO 可 GOT overwrite，Full RELRO 改 return/vtable/hook，Canary 先 leak。
4. 如果有 seccomp，先 `seccomp-tools dump ./bin`，不要默认 `/bin/sh` 可用。
5. 写 pwntools skeleton，先完成本地 leak/ret2win/写入 proof，再远程化。

## Pwn 首轮路线分流

| 首轮信号 | 利用路线判断 |
|---|---|
| 栈溢出 | 偏移、canary、NX、ROP 链。 |
| 格式串 | 任意读/写、GOT/return address、PIE/libc leak。 |
| 堆题 | UAF/double free/tcache poisoning，先确认 glibc 版本。 |
| 保护驱动路线 | RELRO、PIE、Canary、NX、seccomp 决定目标选择。 |

## 常见陷阱

- 不看 checksec 就开始写 ROP。
- 只有 crash 没有 primitive 定义，导致 exploit 反复推倒。
- 远程 libc 不一致却直接套本地 offset。
- seccomp 开启时仍按常规 `system('/bin/sh')` 设计。

## 关联技巧

- [oob-jit-parser-primitives-family.md](oob-jit-parser-primitives-family.md)
- [cross-primitive-escape-and-hybrid-exploit-map.md](cross-primitive-escape-and-hybrid-exploit-map.md)
- [data-interpretation-memory-primitives.md](data-interpretation-memory-primitives.md)
- [format-string.md](format-string.md)
- [heap-fsop-file-structure-attacks.md](heap-fsop-file-structure-attacks.md)
- [heap-houses-unlink-and-tcache.md](heap-houses-unlink-and-tcache.md)
- [overflow-basics.md](overflow-basics.md)
- [heap-uaf-tcache-and-custom-allocator.md](heap-uaf-tcache-and-custom-allocator.md)
- [ret2csu-dynelf-and-shellcode.md](ret2csu-dynelf-and-shellcode.md)
- [stack-pivots-srop-and-seccomp-rop.md](stack-pivots-srop-and-seccomp-rop.md)
- [seccomp-ret2dlresolve-and-runtime-primitives.md](seccomp-ret2dlresolve-and-runtime-primitives.md)
- [runtime-protection-and-tls-exploits.md](runtime-protection-and-tls-exploits.md)
- [linux-kernel-exploit-basics.md](linux-kernel-exploit-basics.md)
- [kaslr-kpti-smep-and-kernel-debugging.md](kaslr-kpti-smep-and-kernel-debugging.md)
- [kernel-uaf-race-and-slab-techniques.md](kernel-uaf-race-and-slab-techniques.md)

- [interpreter-jit-canary-and-integer-exploits.md](interpreter-jit-canary-and-integer-exploits.md)
- [known-cves-and-n-day-exploits.md](known-cves-and-n-day-exploits.md)
- [windows-arm-and-cross-platform-exploits.md](windows-arm-and-cross-platform-exploits.md)

## WP 案例沉淀

本节只抽取 raw WP 中可复用的识别信号和下一跳，不替代原始题解正文。

| Raw WP | 复用信号 | 下一跳 |
|---|---|---|
| [WMCTF2025-aberration-wp](../raw/pwn/WMCTF2025-aberration-wp.md) | ARM64 TrustZone/EL3 SMC handler 存在跨 CPU TOCTOU，先提取 BL31、确认 secure world 调用链和竞态窗口。 | [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md)、[race-condition-and-concurrency-exploits.md](race-condition-and-concurrency-exploits.md) |
| [WMCTF2025-palusimulator-wp](../raw/pwn/WMCTF2025-palusimulator-wp.md) | C++ 异常处理继续执行、负数 size、未初始化变量和 `-O3` 析构优化共同形成泄露、double free、堆溢出和 largebin/tcache 链。 | [heap-uaf-tcache-and-custom-allocator.md](heap-uaf-tcache-and-custom-allocator.md)、[heap-houses-unlink-and-tcache.md](heap-houses-unlink-and-tcache.md) |
| [WMCTF2025-wm-easyker-wp](../raw/pwn/WMCTF2025-wm-easyker-wp.md) | 一次任意读和可控 `rsp` 不直接 ret2dir；先泄露 kernel base，再利用 `/dev/random` 的 `input_pool.hash.buf` 作为概率 ROP 栈。 | [linux-kernel-exploit-basics.md](linux-kernel-exploit-basics.md)、[kaslr-kpti-smep-and-kernel-debugging.md](kaslr-kpti-smep-and-kernel-debugging.md) |
| [WMCTF2025-wm-easynetlink-wp](../raw/pwn/WMCTF2025-wm-easynetlink-wp.md) | Generic Netlink 暴露 UAF 随机写，先用 `bpf_array` 制造 verifier/runtime 差异，再扩展到内核任意读写和页表 patch。 | [kernel-uaf-race-and-slab-techniques.md](kernel-uaf-race-and-slab-techniques.md)、[linux-kernel-exploit-basics.md](linux-kernel-exploit-basics.md) |
| [WMCTF2025-wm-eat-some-qanux-wp](../raw/pwn/WMCTF2025-wm-eat-some-qanux-wp.md) | 自定义 ARM-like VM 的 SP 边界检查错误，直接写 `svc` 被过滤时可让解释器执行内部 SVC opcode。 | [interpreter-jit-canary-and-integer-exploits.md](interpreter-jit-canary-and-integer-exploits.md)、[vm-z3-sandbox-and-game-basics.md](vm-z3-sandbox-and-game-basics.md) |
| [WMCTF2025-wmkpf-wp](../raw/pwn/WMCTF2025-wmkpf-wp.md) | 内核模块管理用户 BPF map fd，利用页表喷射/重叠改写 `modprobe_path` 物理页映射，再触发未知 binfmt 提权。 | [linux-kernel-exploit-basics.md](linux-kernel-exploit-basics.md)、[kaslr-kpti-smep-and-kernel-debugging.md](kaslr-kpti-smep-and-kernel-debugging.md) |
| [ACTF2026-acpu-wp](../raw/pwn/ACTF2026-acpu-wp.md) | RISC-V CPU 仿真器存在 Meltdown 型 forwarding/cache timing 侧信道，先证明非法 load 对后续公开访问的影响。 | [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md) |
| [ACTF2026-agpu-wp](../raw/pwn/ACTF2026-agpu-wp.md) | aarch64 Mali/kbase 环境只有一次 4 字节内核写；改 allocation `pages[0]` 后把 kernel text 物理页 alias 到用户态 mapping 再 patch 提权。 | [linux-kernel-exploit-basics.md](linux-kernel-exploit-basics.md)、[windows-arm-and-cross-platform-exploits.md](windows-arm-and-cross-platform-exploits.md) |
| [ACTF2026-amcu-wp](../raw/pwn/ACTF2026-amcu-wp.md) | MCU 串口输入进入 printf 伪参数，先用 `%n` 建任意写再布置 SRAM 跳板和 I2C shellcode。 | [format-string.md](format-string.md) |
| [ACTF2026-badgate-wp](../raw/pwn/ACTF2026-badgate-wp.md) | Lua userdata `pkt:view()` 不持有请求缓冲区生命周期，`conn:close()` 后形成跨请求悬挂视图；可先泄 `/flag`，再伪造 external string 到 `system`。 | [heap-uaf-tcache-and-custom-allocator.md](heap-uaf-tcache-and-custom-allocator.md)、[cross-primitive-escape-and-hybrid-exploit-map.md](cross-primitive-escape-and-hybrid-exploit-map.md) |
| [moeCTF2022-syscall-wp](../raw/pwn/moeCTF2022-syscall-wp.md) | 程序泄露 PIE 且自带 `syscall`/`/bin/sh` gadget；第二次 `read` 精确发送 59 字节让 `RAX=SYS_execve`。 | [ret2csu-dynelf-and-shellcode.md](ret2csu-dynelf-and-shellcode.md)、[overflow-basics.md](overflow-basics.md) |
| [HGAME2026-adrift-wp](../raw/pwn/HGAME2026-adrift-wp.md) | 整数边界绕过转正逻辑并影响 canary/栈执行，先验证偏移、泄露和 shellcode 条件。 | [overflow-basics.md](overflow-basics.md) |
| [HGAME2026-diary-keeper-wp](../raw/pwn/HGAME2026-diary-keeper-wp.md) | glibc 2.35 off-by-null 伪造合并并泄露 libc/heap，后续不是普通 UAF，而是 `_IO_list_all` + house of obstack。 | [heap-fsop-file-structure-attacks.md](heap-fsop-file-structure-attacks.md) |
| [HGAME2026-gosick-wp](../raw/pwn/HGAME2026-gosick-wp.md) | Rust `rust-gc` 自定义 Trace 导致 sweep 后 content 悬挂引用，先复用释放区改 `Token.uid` 或伪造 `GcBoxHeader` vtable。 | [heap-uaf-tcache-and-custom-allocator.md](heap-uaf-tcache-and-custom-allocator.md) |
| [HGAME2026-heap1sez-wp](../raw/pwn/HGAME2026-heap1sez-wp.md) | 自定义 malloc 注释掉 unlink 的 `fd/bk` 一致性检查，UAF 改 freed chunk 后走 unsafe unlink 和隐藏 `hook`。 | [heap-houses-unlink-and-tcache.md](heap-houses-unlink-and-tcache.md) |
| [HGAME2026-ionostream-wp](../raw/pwn/HGAME2026-ionostream-wp.md) | UAF/tcache/fastbin 与 exit function 链表目标组合，先确认 glibc、safe-linking 和任意写落点。 | [heap-uaf-tcache-and-custom-allocator.md](heap-uaf-tcache-and-custom-allocator.md) |
| [HGAME2026-producer-and-comsumer-wp](../raw/pwn/HGAME2026-producer-and-comsumer-wp.md) | 生产者检查槽位和真正写入不在同一临界区，消费者可在 sleep 窗口改共享索引，最终越界写覆盖栈并 `leave; ret` 迁移。 | [race-condition-and-concurrency-exploits.md](race-condition-and-concurrency-exploits.md)、[stack-pivots-srop-and-seccomp-rop.md](stack-pivots-srop-and-seccomp-rop.md) |
| [HGAME2026-steins-gate-wp](../raw/pwn/HGAME2026-steins-gate-wp.md) | 逐字节比较错误会在不同位置 panic，PIE 仍保留稳定页内低 12 位偏移；用崩溃地址作为比较进度 oracle 爆破输入。 | [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md)、[interpreter-jit-canary-and-integer-exploits.md](interpreter-jit-canary-and-integer-exploits.md) |
| [LilacCTF2026-bytezoo-wp](../raw/pwn/LilacCTF2026-bytezoo-wp.md) | 受限执行环境需要构造运行时读写/执行 primitive，先验证 mprotect、memmove 和二阶段 shellcode。 | [seccomp-ret2dlresolve-and-runtime-primitives.md](seccomp-ret2dlresolve-and-runtime-primitives.md) |
| [LilacCTF2026-chuantongxiangyan-wp](../raw/pwn/LilacCTF2026-chuantongxiangyan-wp.md) | 格式化字符串可控，先定义泄露、写入粒度和目标地址，再选择 GOT/返回地址/短写链。 | [format-string.md](format-string.md) |
| [LilacCTF2026-elk-wp](../raw/pwn/LilacCTF2026-elk-wp.md) | 嵌入式 JS 引擎对象/属性和 NaN-boxing 形成调用原语，先定义伪造对象到 C 函数调用的 primitive。 | [oob-jit-parser-primitives-family.md](oob-jit-parser-primitives-family.md) |
| [LilacCTF2026-gate-way-wp](../raw/pwn/LilacCTF2026-gate-way-wp.md) | Hexagon 架构栈溢出要先理解 SP/FP/LR 和 `dealloc_return`，用伪栈帧装载 R0/R1/R2/R6 后 `trap0(#1)`。 | [windows-arm-and-cross-platform-exploits.md](windows-arm-and-cross-platform-exploits.md)、[overflow-basics.md](overflow-basics.md) |
| [LilacCTF2026-na1vm-wp](../raw/pwn/LilacCTF2026-na1vm-wp.md) | 自定义 VM 任务队列 OOB 覆盖 `head/tail/size` 后控制寄存器和栈基址；任意写还要恢复 glibc `__exit_funcs` pointer mangling key。 | [oob-jit-parser-primitives-family.md](oob-jit-parser-primitives-family.md)、[runtime-protection-and-tls-exploits.md](runtime-protection-and-tls-exploits.md) |
| [LilacCTF2026-trustsql-plus-wp](../raw/pwn/LilacCTF2026-trustsql-plus-wp.md) | 上传 SQLite 库不是普通 SQLi，关键是 FTS4 `compress/uncompress` 函数名拼入内部顶层 SQL，绕过 `trusted_schema=0` 后 `writefile+load_extension`。 | [known-cves-and-n-day-exploits.md](known-cves-and-n-day-exploits.md)、[sqli-upload-deser-and-command-rce.md](sqli-upload-deser-and-command-rce.md) |
| [NCTF2026-checkin-wp](../raw/pwn/NCTF2026-checkin-wp.md) | `read` 后直接 `printf(buf)` 再 `_exit`，核心是格式串泄露/短写，利用 `_exit` GOT 或栈低字节把控制流循环起来。 | [format-string.md](format-string.md) |
| [NCTF2026-ezheap-wp](../raw/pwn/NCTF2026-ezheap-wp.md) | 常规 IO 触发面被拿掉后仍可 largebin attack 改 `mp_.tcache_bins`，再用 tcache poisoning 做 AAR/AAW。 | [heap-houses-unlink-and-tcache.md](heap-houses-unlink-and-tcache.md) |
| [NCTF2026-vfs-stack-wp](../raw/pwn/NCTF2026-vfs-stack-wp.md) | VFS/内核对象或驱动攻击面，先确认 ioctl、对象生命周期和提权原语。 | [linux-kernel-exploit-basics.md](linux-kernel-exploit-basics.md) |
| [RCTF2025-bbox-wp](../raw/pwn/RCTF2025-bbox-wp.md) | PCI/MMIO 设备 `merge` 未检查合并 size，先把 block OOB read/write 转成设备/宿主侧指针覆盖和可控 `rdi` 调用。 | [cross-primitive-escape-and-hybrid-exploit-map.md](cross-primitive-escape-and-hybrid-exploit-map.md) |
| [RCTF2025-mstr-wp](../raw/pwn/RCTF2025-mstr-wp.md) | 运行时字符串对象重叠形成越界读写并最终打 FSOP，先确认对象布局、libc 和 FILE 结构目标。 | [heap-fsop-file-structure-attacks.md](heap-fsop-file-structure-attacks.md) |
| [RCTF2025-no-check-wasm-wp](../raw/pwn/RCTF2025-no-check-wasm-wp.md) | V8/WASM 类型校验缺陷允许 `externref` 与 `i64` 互转，先构造 `addrOf/fakeObj`，再泄露栈和 RWX 页写 shellcode。 | [oob-jit-parser-primitives-family.md](oob-jit-parser-primitives-family.md)、[interpreter-jit-canary-and-integer-exploits.md](interpreter-jit-canary-and-integer-exploits.md) |
| [RCTF2025-only-rev-wp](../raw/pwn/RCTF2025-only-rev-wp.md) | syscall stub 或运行时寄存器残留提供读写/执行入口，先验证可控寄存器、可写段和 shellcode 通道。 | [seccomp-ret2dlresolve-and-runtime-primitives.md](seccomp-ret2dlresolve-and-runtime-primitives.md) |
| [RCTF2025-only-wp](../raw/pwn/RCTF2025-only-wp.md) | 浮点/机器码输入进入 syscall stub，先确认可执行内存、read primitive 和二阶段 shellcode。 | [seccomp-ret2dlresolve-and-runtime-primitives.md](seccomp-ret2dlresolve-and-runtime-primitives.md) |
| [RCTF2025-rd-wp](../raw/pwn/RCTF2025-rd-wp.md) | 未初始化 task 指针可踩 stdout，先泄露 heap/libc，再构造 fake stdout / fake FILE 控制执行流。 | [heap-fsop-file-structure-attacks.md](heap-fsop-file-structure-attacks.md) |
| [Spirit2026-5-large-wp](../raw/pwn/Spirit2026-5-large-wp.md) | glibc 2.39 只给 largebin 尺寸堆块，UAF 泄露后用 largebin attack 改 `g_f`，绕过 SHSTK/IBT/GOT 路线。 | [heap-houses-unlink-and-tcache.md](heap-houses-unlink-and-tcache.md) |
| [SUCTF2026-BoxWP](../raw/pwn/SUCTF2026-BoxWP.md) | J2V8 内嵌 V8 9.3.345.11，先排除 Java 沙箱逃逸，再用 CVE-2021-38003 `JSON.stringify` hole 做 OOB/addrof/RWX wasm。 | [oob-jit-parser-primitives-family.md](oob-jit-parser-primitives-family.md) |
| [SUCTF2026-Chronos_Ring_SUCTF2026-Chronos_Ring1WP](../raw/pwn/SUCTF2026-Chronos_Ring_SUCTF2026-Chronos_Ring1WP.md) | 内核模块 ioctl 先绕过与 `kfree` KASLR 地址相关的 key，再把匿名 buffer 提交到 `/tmp/job` page cache，借 root 定时任务提权。 | [linux-kernel-exploit-basics.md](linux-kernel-exploit-basics.md)、[kaslr-kpti-smep-and-kernel-debugging.md](kaslr-kpti-smep-and-kernel-debugging.md) |
| [SUCTF2026-evbufferWP](../raw/pwn/SUCTF2026-evbufferWP.md) | TCP/UDP `inet_pton` 后全局溢出伪造 libevent `bufferevent/evbuffer` callback，seccomp 下栈迁移到 ORW。 | [seccomp-ret2dlresolve-and-runtime-primitives.md](seccomp-ret2dlresolve-and-runtime-primitives.md)、[cross-primitive-escape-and-hybrid-exploit-map.md](cross-primitive-escape-and-hybrid-exploit-map.md) |
| [SUCTF2026-EzRouterWP](../raw/pwn/SUCTF2026-EzRouterWP.md) | Web 鉴权、CGI IPC、固件堆溢出和 shellcode 串成跨层链，先固定 Web 到 native 的数据流。 | [cross-primitive-escape-and-hybrid-exploit-map.md](cross-primitive-escape-and-hybrid-exploit-map.md) |
| [SUCTF2026-minivfsWP](../raw/pwn/SUCTF2026-minivfsWP.md) | VFS 风格接口背后是 glibc 2.41 largebin/off-by-null/overlap 堆利用，先稳定 libc/heap leak 和 chunk 布局。 | [heap-houses-unlink-and-tcache.md](heap-houses-unlink-and-tcache.md) |
| [VNCTF2026-eat-some-ai-wp](../raw/pwn/VNCTF2026-eat-some-ai-wp.md) | 文字游戏商人购买数量存在 32 位整数溢出，先把积分反向增加到可执行命令状态。 | [interpreter-jit-canary-and-integer-exploits.md](interpreter-jit-canary-and-integer-exploits.md) |
| [VNCTF2026-ezphp-wp](../raw/pwn/VNCTF2026-ezphp-wp.md) | PHP 扩展 off-by-one 扩大 `vh_edit` 范围后覆盖 `comment` 指针字段；用 `/proc/self/maps` 泄露基址并把 `efree@GOT` 改成 `system`。 | [cross-primitive-escape-and-hybrid-exploit-map.md](cross-primitive-escape-and-hybrid-exploit-map.md)、[heap-uaf-tcache-and-custom-allocator.md](heap-uaf-tcache-and-custom-allocator.md) |
| [VNCTF2026-real-world-wp](../raw/pwn/VNCTF2026-real-world-wp.md) | 原样 Rsync 3.3.0 服务，先用 CVE-2024-12085 未初始化数据泄漏定位地址，再用 CVE-2024-12084 checksum 堆溢出转任意写。 | [known-cves-and-n-day-exploits.md](known-cves-and-n-day-exploits.md)、[heap-uaf-tcache-and-custom-allocator.md](heap-uaf-tcache-and-custom-allocator.md) |
| [VNCTF2026-recode-wp](../raw/pwn/VNCTF2026-recode-wp.md) | Protocol Buffers 对象 UAF 进入 tcache poisoning、任意读写和栈 ROP，先确认对象生命周期和 libc/heap leak。 | [heap-uaf-tcache-and-custom-allocator.md](heap-uaf-tcache-and-custom-allocator.md) |
| [VNCTF2026-vm-syscall-wp](../raw/pwn/VNCTF2026-vm-syscall-wp.md) | VM `case4` 把 4 个寄存器映射到 syscall，但执行前清零 `r8/r9/r10`；绕过常规 `mmap`，用 `shmget/shmat` 放 `/bin/sh` 后 `execve`。 | [seccomp-ret2dlresolve-and-runtime-primitives.md](seccomp-ret2dlresolve-and-runtime-primitives.md)、[vm-z3-sandbox-and-game-basics.md](vm-z3-sandbox-and-game-basics.md) |
| [D3CTF2019-babyrop-wp](../raw/pwn/D3CTF2019-babyrop-wp.md) | 基础 ROP/栈溢出路线，先确认偏移、保护和 ret2libc/ROP 链条件。 | [overflow-basics.md](overflow-basics.md) |
| [D3CTF2019-basic-basic-parser-wp](../raw/pwn/D3CTF2019-basic-basic-parser-wp.md) | parser 对象生命周期和 UAF/tcache 行为可控，先定义堆读写 primitive。 | [heap-uaf-tcache-and-custom-allocator.md](heap-uaf-tcache-and-custom-allocator.md) |
| [D3CTF2019-ezfile-wp](../raw/pwn/D3CTF2019-ezfile-wp.md) | tcache double free 配合 FILE fileno 和受限 ORW，先复用已有 FD 而不是追求 shell。 | [heap-fsop-file-structure-attacks.md](heap-fsop-file-structure-attacks.md) |
| [D3CTF2019-knote-v1-v2-wp](../raw/pwn/D3CTF2019-knote-v1-v2-wp.md) | 内核 note 对象 UAF/堆布局是主线，先确认 slab 对象复用和提权目标。 | [kernel-uaf-race-and-slab-techniques.md](kernel-uaf-race-and-slab-techniques.md) |
| [D3CTF2019-lonely-observer-wp](../raw/pwn/D3CTF2019-lonely-observer-wp.md) | 堆对象 UAF 和观察/泄露接口组合，先稳定 leak 再转任意写。 | [heap-uaf-tcache-and-custom-allocator.md](heap-uaf-tcache-and-custom-allocator.md) |
| [D3CTF2019-new-heap-wp](../raw/pwn/D3CTF2019-new-heap-wp.md) | 堆管理器和 tcache/house 技巧是主线，先确认 glibc 版本、bin 状态和目标 chunk。 | [heap-houses-unlink-and-tcache.md](heap-houses-unlink-and-tcache.md) |
| [D3CTF2019-unprintablev-wp](../raw/pwn/D3CTF2019-unprintablev-wp.md) | stdout 被关闭的格式化字符串题，先恢复输出或改 FILE 描述符再 leak/ORW。 | [format-string.md](format-string.md) |
| [D3CTF2021-狡兔三窟-wp](../raw/pwn/D3CTF2021-狡兔三窟-wp.md) | UAF/tcache 触发多阶段堆利用，先确认 dangling pointer 和重分配窗口。 | [heap-uaf-tcache-and-custom-allocator.md](heap-uaf-tcache-and-custom-allocator.md) |
| [D3CTF2021-d3dev-d3dev-revenge-wp](../raw/pwn/D3CTF2021-d3dev-d3dev-revenge-wp.md) | QEMU 虚拟设备 MMIO/PMIO 越界覆盖函数指针，先建 guest 到 host 的设备状态 primitive。 | [cross-primitive-escape-and-hybrid-exploit-map.md](cross-primitive-escape-and-hybrid-exploit-map.md) |
| [D3CTF2021-deterministic-heap-wp](../raw/pwn/D3CTF2021-deterministic-heap-wp.md) | 确定性堆布局和对象复用是主线，先固定 allocator 行为和可控对象落点。 | [heap-uaf-tcache-and-custom-allocator.md](heap-uaf-tcache-and-custom-allocator.md) |
| [D3CTF2021-easy-chrome-full-chain-wp](../raw/pwn/D3CTF2021-easy-chrome-full-chain-wp.md) | Chrome/V8 full chain 需要先拿浏览器内 read/write/call primitive，再串沙箱逃逸。 | [oob-jit-parser-primitives-family.md](oob-jit-parser-primitives-family.md) |
| [D3CTF2021-hackphp-wp](../raw/pwn/D3CTF2021-hackphp-wp.md) | PHP 扩展对象 UAF 可伪造 zend_closure，先泄露可控 string 地址并布置对象。 | [heap-uaf-tcache-and-custom-allocator.md](heap-uaf-tcache-and-custom-allocator.md) |
| [D3CTF2021-liproll-wp](../raw/pwn/D3CTF2021-liproll-wp.md) | FG-KASLR 下内核栈溢出转任意读写，先泄露代码段或搜索 cred/ROP 目标。 | [kaslr-kpti-smep-and-kernel-debugging.md](kaslr-kpti-smep-and-kernel-debugging.md) |
| [D3CTF2021-real-vmpwn-wp](../raw/pwn/D3CTF2021-real-vmpwn-wp.md) | 自定义 VM/解释器对象漏洞，先定义 VM 指令面到宿主内存 primitive。 | [oob-jit-parser-primitives-family.md](oob-jit-parser-primitives-family.md) |
| [D3CTF2021-truth-wp](../raw/pwn/D3CTF2021-truth-wp.md) | 源码检查被 LLVM UB 优化折叠，先对比编译产物确认真实边界检查和堆溢出。 | [interpreter-jit-canary-and-integer-exploits.md](interpreter-jit-canary-and-integer-exploits.md) |
| [D3CTF2022-d3bpf-v2-wp](../raw/pwn/D3CTF2022-d3bpf-v2-wp.md) | eBPF/verifier 或内核 JIT 约束是主线，先确认 verifier 绕过和内核读写 primitive。 | [linux-kernel-exploit-basics.md](linux-kernel-exploit-basics.md) |
| [D3CTF2022-d3bpf-wp](../raw/pwn/D3CTF2022-d3bpf-wp.md) | eBPF 内核攻击面，先还原 verifier 约束、map 状态和可控内核指针。 | [linux-kernel-exploit-basics.md](linux-kernel-exploit-basics.md) |
| [D3CTF2022-d3fuse-wp](../raw/pwn/D3CTF2022-d3fuse-wp.md) | FUSE/内核交互导致对象生命周期漏洞，先确认用户态文件系统回调和内核对象复用。 | [linux-kernel-exploit-basics.md](linux-kernel-exploit-basics.md) |
| [D3CTF2022-d3guard-wp](../raw/pwn/D3CTF2022-d3guard-wp.md) | 内核驱动/保护逻辑和 shellcode 路线，先确认 ioctl、映射和执行边界。 | [linux-kernel-exploit-basics.md](linux-kernel-exploit-basics.md) |
| [D3CTF2022-d3kheap-wp](../raw/pwn/D3CTF2022-d3kheap-wp.md) | 内核堆 UAF 与 msg_msg/pipe_buffer 等对象复用，先固定 slab 布局和提权写点。 | [kernel-uaf-race-and-slab-techniques.md](kernel-uaf-race-and-slab-techniques.md) |
| [D3CTF2022-smart-calculator-wp](../raw/pwn/D3CTF2022-smart-calculator-wp.md) | 内核模块/计算器接口产生提权 primitive，先确认 ioctl 输入面和 cred 修改路线。 | [linux-kernel-exploit-basics.md](linux-kernel-exploit-basics.md) |
| [D3CTF2023-d3kcache-wp](../raw/pwn/D3CTF2023-d3kcache-wp.md) | 页级 UAF、buddy/slab/pipe_buffer 复用是主线，先把 page 到 pipe 的重分配稳定化。 | [kernel-uaf-race-and-slab-techniques.md](kernel-uaf-race-and-slab-techniques.md) |
| [D3CTF2023-d3op-wp](../raw/pwn/D3CTF2023-d3op-wp.md) | OpenWrt/AArch64 rootfs 审计到 ubus RPC 栈溢出，先还原固件服务和跨架构 ROP 条件。 | [windows-arm-and-cross-platform-exploits.md](windows-arm-and-cross-platform-exploits.md)、[overflow-basics.md](overflow-basics.md) |
| [D3CTF2023-d3trustedhttpd-wp](../raw/pwn/D3CTF2023-d3trustedhttpd-wp.md) | HTTPd 到 OP-TEE TA 的 RPC/secure storage/CA 栈溢出混合链，先定义 REE/TEE 数据流。 | [cross-primitive-escape-and-hybrid-exploit-map.md](cross-primitive-escape-and-hybrid-exploit-map.md) |
| [D3CTF2023-realesxi-wp](../raw/pwn/D3CTF2023-realesxi-wp.md) | ESXi vmx 沙箱逃逸需要 ticket 伪造和 settingsd TOCTOU，先建服务 API 与文件写 primitive。 | [cross-primitive-escape-and-hybrid-exploit-map.md](cross-primitive-escape-and-hybrid-exploit-map.md) |
| [D3CTF2025-d3cgi-wp](../raw/pwn/D3CTF2025-d3cgi-wp.md) | FastCGI 1-day 内存破坏依赖 record 长度和堆布局稳定性，先复现补丁差异和二进制请求。 | [known-cves-and-n-day-exploits.md](known-cves-and-n-day-exploits.md) |
| [D3CTF2025-d3kheap2-wp](../raw/pwn/D3CTF2025-d3kheap2-wp.md) | 新版内核堆题继续围绕 UAF/page/slab 复用，先确认对象释放窗口和可控映射。 | [kernel-uaf-race-and-slab-techniques.md](kernel-uaf-race-and-slab-techniques.md) |
| [D3CTF2025-d3kshrm-d3kshrm-revenge-wp](../raw/pwn/D3CTF2025-d3kshrm-d3kshrm-revenge-wp.md) | 共享内存 mmap fault 下标越界可映射相邻 struct page，先稳定伪造页表和内核任意映射。 | [kernel-uaf-race-and-slab-techniques.md](kernel-uaf-race-and-slab-techniques.md) |

## 原始资料
- [pwn-first-pass-red-flags-and-protections.md](../raw/pwn/pwn-first-pass-red-flags-and-protections.md)
