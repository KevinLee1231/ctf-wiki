---
type: family
tags: [pwn, family, heap, tcache, unlink, house, allocator]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/heap-houses-unlink-and-tcache.md
  - ../raw/pwn/WMCTF2025-palusimulator-wp.md
  - ../raw/pwn/D3CTF2019-new-heap-wp.md
updated: 2026-07-06
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

## 合并与拆分结论

本页应为 family。它与 [heap-uaf-tcache-and-custom-allocator.md](heap-uaf-tcache-and-custom-allocator.md) 分工：本页偏 metadata/bin/house 路线，后者偏 UAF/对象生命周期和 tcache primitive 的形成。当前不拆 House 系列小页，因为 raw 仍以速查集合为主。

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
