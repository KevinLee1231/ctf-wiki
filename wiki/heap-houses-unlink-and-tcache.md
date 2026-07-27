---
type: family
tags: [pwn, family, heap, tcache, unlink, house, allocator]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/heap-houses-unlink-and-tcache.md
  - ../raw/pwn/WMCTF2025-palusimulator-wp.md
  - ../raw/pwn/D3CTF2019-new-heap-wp.md
updated: 2026-07-27
---

# Heap Houses, Unlink and Tcache

## 作用边界

本页是 heap metadata 和 allocator 路线 family，覆盖 House of Apple/Orange/Spirit/Lore/Force/Einherjar、classic unlink、tcache stashing unlink、largebin 写、custom allocator、talloc、musl meta/atexit 和 heap grooming。

它不是单一 heap technique。首轮要先判断 libc/allocator、漏洞 primitive、chunk 大小、bin 状态、safe-linking、可泄露地址和最终落点，再决定走 unlink、tcache poisoning、FSOP、largebin、top chunk、custom allocator 还是 runtime handler。

## 识别信号

- 有堆溢出、off-by-one null、UAF、double free、越界写、可控 free 顺序或自定义 allocator。
- 可影响 chunk header、tcache fd、unsorted/largebin metadata、top chunk、FILE 结构、wide data、atexit handler 或 allocator meta。
- exploit 依赖 libc/glibc 版本、safe-linking、tcache 数量、chunk alignment、consolidation 和 heap leak。

## 最小证据

- 确认 allocator 和 libc 版本：glibc/musl/talloc/custom，以及 tcache/safe-linking 是否存在。
- 画出 allocation/free 序列、chunk size、bin 状态、重叠关系和可写字段。
- 至少证明一个 primitive：overlap、arbitrary write、tcache poisoning、largebin write、FILE 控制或 hook/handler 覆盖。
- 记录最终触发点：malloc/free/exit/flush/setcontext/atexit/ROP。

## 首轮路由

| 证据形态 | 首轮判断 | 下一跳 |
|---|---|---|
| off-by-one null、`PREV_INUSE` 可清 | 优先检查 House of Einherjar 或 backward consolidation 造成 overlap | [heap-uaf-tcache-and-custom-allocator.md](heap-uaf-tcache-and-custom-allocator.md) |
| tcache fd、double free、safe-linking | 先拿 heap leak 并计算 mangled fd，再做 tcache poisoning 或 stashing unlink | [runtime-protection-and-tls-exploits.md](runtime-protection-and-tls-exploits.md) |
| unsorted/largebin metadata 可控 | 判断是 libc leak、global_max_fast/stack variable 写，还是扩大后续 read | [heap-fsop-file-structure-attacks.md](heap-fsop-file-structure-attacks.md) |
| FILE/wide data/House of Apple/Orange | 转 FSOP 和 runtime 保护页，优先核对 glibc 版本字段 | [heap-fsop-file-structure-attacks.md](heap-fsop-file-structure-attacks.md), [runtime-protection-and-tls-exploits.md](runtime-protection-and-tls-exploits.md) |
| top chunk size 可控 | 检查 House of Force/Orange 的 size、page boundary 和 sysmalloc 条件 | [pwn-tooling.md](pwn-tooling.md) |
| musl meta、atexit、custom allocator、talloc | 先恢复 allocator 元数据和回调触发点，不要套 glibc chunk 模型 | [runtime-protection-and-tls-exploits.md](runtime-protection-and-tls-exploits.md) |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [WMCTF2025-palusimulator-wp](../raw/pwn/WMCTF2025-palusimulator-wp.md) | 异常处理产生可泄露的 0x90 chunk 后，先构造小 chunk 进入 0x90 tcache，再用 largebin attack 写栈上 `read` 长度变量，最后 tcache fd 指向栈布置 ORW ROP。 |
| [D3CTF2019-new-heap-wp](../raw/pwn/D3CTF2019-new-heap-wp.md) | glibc 2.29 tcache key 检查、count 限制和 cross-bin overlap 共同决定路线；利用点不只是 double free，而是通过 consolidation/overlap 控制 tcache struct、stdout 和 hook 落点。 |
| [HGAME2026-heap1sez-wp](../raw/pwn/HGAME2026-heap1sez-wp.md) | 自定义 malloc 注释掉 unlink 的 `fd/bk` 一致性检查，UAF 改 freed chunk 后走 unsafe unlink 和隐藏 `hook`。 |
| [NCTF2026-ezheap-wp](../raw/pwn/NCTF2026-ezheap-wp.md) | 常规 IO 触发面被拿掉后仍可 largebin attack 改 `mp_.tcache_bins`，再用 tcache poisoning 做 AAR/AAW。 |
| [Spirit2026-5-large-wp](../raw/pwn/Spirit2026-5-large-wp.md) | glibc 2.39 只给 largebin 尺寸堆块，UAF 泄露后用 largebin attack 改 `g_f`，绕过 SHSTK/IBT/GOT 路线。 |
| [SUCTF2026-minivfsWP](../raw/pwn/SUCTF2026-minivfsWP.md) | VFS 风格接口背后是 glibc 2.41 largebin/off-by-null/overlap 堆利用，先稳定 libc/heap leak 和 chunk 布局。 |
| [D3CTF2024-PwnShell-wp](../raw/pwn/D3CTF2024-PwnShell-wp.md) | 本题利用链由三个独立环节组成：off-by-null 改写相邻堆块元数据、伪造空闲链表指针取得任意地址写、通过 `/proc/self/maps` 消除 ASLR 影响。最后选择覆盖 `efree@GOT`，是因为扩展会在删除对象时把可控内容指针直接传给 `efree`，函数参数天然满足命令执行入口的调用约定。 |
| [UMDCTF2022-the-show-must-go-on-wp](../raw/pwn/UMDCTF2022-the-show-must-go-on-wp.md) | 题名虽然提示 tcache，但这里无需伪造链表指针。关键是让新申请精确复用已释放的 `0x90` chunk，再利用越界写覆盖物理相邻对象。分析这类题时，应分别计算“请求大小到 chunk 大小的归一化”和“目标字段相对当前用户区的总距离”，不能只看源码中的数组长度。 |
| [UMDCTF2024-worm-eat-worm-wp](../raw/pwn/UMDCTF2024-worm-eat-worm-wp.md) | 本题的关键是 C++ 所有权错误，而不是 vector 越界：自赋值把已释放指针保留在对象中，`Get` 泄露 tcache 链，`Rename` 写入伪造链头。Full RELRO 只保护主程序并不代表所有已加载共享库都没有可写 GOT；这里选择 libstdc++ 的 `free@GOT - 0x10`，还能让被劫持的 `free` 同时拿到可直接执行的 `/bin/sh` 参数。 |
| [UMDCTF2025-one-write-wp](../raw/pwn/UMDCTF2025-one-write-wp.md) | 固定悬空指针可随分配阶段覆盖不同 allocator 元数据：unsorted bin 泄漏 libc、safe-linked tcache 泄漏 heap、poison 到 `__libc_argv` 泄漏 stack 和 PIE，最后借 smallbin tcache stashing 写回全局指针。 |
| [UMDCTF2026-velvet-table-wp](../raw/pwn/UMDCTF2026-velvet-table-wp.md) | 这道题把关键条件都写进了源码：tcache 满后残留的悬空指针、可逆掩码、栈地址派生的 marker、栈上伪 smallbin 块以及带标签的函数指针。正确分析顺序是先恢复索引和掩码，再建立 UAF 泄露/写原语，最后按实际 glibc 版本完成 smallbin 到栈的重叠；仅覆盖函数指针而不同时更新标签不会成功。 |
| [WMCTF2023-blindless-wp](../raw/pwn/WMCTF2023-blindless-wp.md) | 利用 `house of blindless` 改写动态链接器结构，把退出路径转成 `system("/bin/sh")`；Pwn 题只给有限写 primitive，且目标是动态链接 ELF 时，应检查 `.dynamic`、`link_map`、`DT_FINI`、`DT_FINI_ARRAY` 等退出时会被访问的结构。 |
| [WMCTF2023-roguegate-wp](../raw/pwn/WMCTF2023-roguegate-wp.md) | Windows NT 堆溢出构造 `unlink`，再配合迷宫奖励写入和 `__crt_lowio_handle_data.osfile` 绕过 `0x1a` 输入限制，最终建立任意读写。 |

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| chunk header、unlink、unsorted/largebin 链表可被破坏 | [heap-metadata-and-bin-list-corruption.md](heap-metadata-and-bin-list-corruption.md) |
| UAF/double free/tcache freelist 可转任意分配 | [uaf-object-reuse-and-tcache-poisoning.md](uaf-object-reuse-and-tcache-poisoning.md) |
| FILE/exit/TLS 清理结构是最终触发目标 | [file-structure-and-exit-handler-control-flow.md](file-structure-and-exit-handler-control-flow.md) |

## 合并与拆分结论

本页应为 family。它与 [heap-uaf-tcache-and-custom-allocator.md](heap-uaf-tcache-and-custom-allocator.md) 分工：本页偏 metadata/bin/house 路线，后者偏 UAF/对象生命周期和 tcache primitive 的形成。metadata/bin、UAF/tcache 和 FILE/exit 落点已拆为三个 technique；具体 House 名称继续作为版本相关变体保留在本页。

## 常见陷阱

- 不确认 glibc 版本就套 House of Apple/Orange 字段。
- safe-linking 下忘记 mangling 或 heap base。
- 只构造 overlap，没有规划最终触发点。
- largebin 写目标不满足对齐或写入时机，导致目标值不可控。
- custom allocator 题误套 glibc chunk header。

## 关联技巧

- [heap-uaf-tcache-and-custom-allocator.md](heap-uaf-tcache-and-custom-allocator.md)
- [heap-fsop-file-structure-attacks.md](heap-fsop-file-structure-attacks.md)
- [runtime-protection-and-tls-exploits.md](runtime-protection-and-tls-exploits.md)
- [stack-pivots-srop-and-seccomp-rop.md](stack-pivots-srop-and-seccomp-rop.md)
- [pwn-tooling.md](pwn-tooling.md)

## 原始资料

- [heap-houses-unlink-and-tcache.md](../raw/pwn/heap-houses-unlink-and-tcache.md)
- [WMCTF2025-palusimulator-wp](../raw/pwn/WMCTF2025-palusimulator-wp.md)
- [D3CTF2019-new-heap-wp](../raw/pwn/D3CTF2019-new-heap-wp.md)
