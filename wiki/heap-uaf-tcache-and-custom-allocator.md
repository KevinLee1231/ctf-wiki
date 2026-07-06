---
type: family
tags: [pwn, family, heap, uaf, tcache, custom-allocator]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/heap-uaf-tcache-and-custom-allocator.md
  - ../raw/pwn/WMCTF2025-palusimulator-wp.md
  - ../raw/pwn/D3CTF2019-basic-basic-parser-wp.md
  - ../raw/pwn/D3CTF2019-new-heap-wp.md
  - ../raw/pwn/D3CTF2021-deterministic-heap-wp.md
updated: 2026-07-06
---

# Heap UAF, Tcache and Custom Allocators

## 作用边界

本页是用户态 heap 生命周期与 allocator primitive family。它处理 UAF、double free、未初始化 chunk 残留、tcache poisoning、fastbin/tcache cross-bin、对象相邻覆盖、C++ 异常路径、custom allocator unlink 和函数指针/虚表落点。

它与 [heap-houses-unlink-and-tcache.md](heap-houses-unlink-and-tcache.md) 不合并：本页先回答“对象生命周期如何形成 primitive”，后者更偏 house/unlink/bin metadata 技巧选择。

## 识别信号

- 菜单题或对象管理题存在 add/edit/delete/show，指针释放后仍可读写或重复释放。
- glibc/tcache/fastbin/unsorted bin 行为影响可利用性，或源码包含 custom allocator。
- 崩溃点在虚表、函数指针、GOT、hook、IO FILE、exit function、对象方法或相邻 struct。
- 异常处理、引用计数、隐藏菜单、长度截断、off-by-null、LSB-only overwrite 导致对象状态和源码防护不一致。

## 最小证据

- glibc 版本、allocator 类型、chunk size、bin 状态和是否启用 safe-linking。
- 明确 primitive：leak、double free、tcache fd 控制、overlap、partial overwrite、函数指针增量写。
- 已证明目标落点可达：GOT、vtable、`__free_hook`、exit function、FILE 结构、栈地址或对象方法表。
- 对 custom allocator，要先还原 metadata 格式和 unlink/check 条件。

## 路由表

| 证据 | 先验证 | 下一跳 |
|---|---|---|
| UAF read/show | 泄露 heap/libc/PIE 哪个地址，是否需要 unsorted bin | 建 leak 再决定写落点 |
| double free / tcache poisoning | safe-linking key、size class 和 fd 可控性 | 写 hook、GOT、stack 或对象指针 |
| off-by-null / backward consolidation | 前后 chunk size、prev_inuse 和可合并范围 | 转 [heap-houses-unlink-and-tcache.md](heap-houses-unlink-and-tcache.md) |
| adjacent struct overflow | 相邻对象是否有函数指针/长度/索引 | 优先覆盖业务指针或权限字段 |
| C++ exception path | `bad_alloc`、析构、优化和局部变量残留是否改变状态 | 对照编译产物而不是只看源码 |
| cross-bin promotion | tcache/fastbin/unsorted 是否可跨 bin 重用 | 固定释放顺序和 size class |
| custom allocator unlink | metadata 检查、前后指针和写入目标 | 复现 unlink 公式，再选择 GOT/指针表 |
| IO FILE 作为落点 | stdout/stdin/FILE 被堆 primitive 影响 | 转 [heap-fsop-file-structure-attacks.md](heap-fsop-file-structure-attacks.md) |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [WMCTF2025-palusimulator-wp](../raw/pwn/WMCTF2025-palusimulator-wp.md) | 负数 size 触发异常后没有 return，析构函数在 `-O3` 下不再设置 `FREEDBUF`，结合残留指针形成 double free、泄露和 tcache fd 劫持。 |
| [D3CTF2019-basic-basic-parser-wp](../raw/pwn/D3CTF2019-basic-basic-parser-wp.md) | C++ parser 错误恢复路径 `delete` 了已插入全局列表的 `Process` 对象；ASan 定位 UAF 后，可覆盖字符串/容器成员泄露并推进到 double free/tcache attack。 |
| [D3CTF2019-new-heap-wp](../raw/pwn/D3CTF2019-new-heap-wp.md) | glibc 2.29 下 tcache double-free 检查和 malloc 次数限制同时存在；借 `stdin/getchar` 额外大 chunk 触发 `malloc_consolidate`，再做 overlap、stdout leak 和 `__free_hook`。 |
| [D3CTF2021-deterministic-heap-wp](../raw/pwn/D3CTF2021-deterministic-heap-wp.md) | Windows NT Heap/LFH 中利用 bucket、subsegment 和 `CachedItems` 的可预测行为稳定重占释放块，把低概率 UAF 转成可复现对象重叠。 |
| [ACTF2026-badgate-wp](../raw/pwn/ACTF2026-badgate-wp.md) | Lua userdata `pkt:view()` 不持有请求缓冲区生命周期，`conn:close()` 后形成跨请求悬挂视图；可先泄 `/flag`，再伪造 external string 到 `system`。 |
| [D3CTF2019-lonely-observer-wp](../raw/pwn/D3CTF2019-lonely-observer-wp.md) | 堆对象 UAF 和观察/泄露接口组合，先稳定 leak 再转任意写。 |
| [D3CTF2021-狡兔三窟-wp](../raw/pwn/D3CTF2021-狡兔三窟-wp.md) | UAF/tcache 触发多阶段堆利用，先确认 dangling pointer 和重分配窗口。 |
| [D3CTF2021-hackphp-wp](../raw/pwn/D3CTF2021-hackphp-wp.md) | PHP 扩展对象 UAF 可伪造 zend_closure，先泄露可控 string 地址并布置对象。 |
| [HGAME2026-gosick-wp](../raw/pwn/HGAME2026-gosick-wp.md) | Rust `rust-gc` 自定义 Trace 导致 sweep 后 content 悬挂引用，先复用释放区改 `Token.uid` 或伪造 `GcBoxHeader` vtable。 |
| [HGAME2026-ionostream-wp](../raw/pwn/HGAME2026-ionostream-wp.md) | UAF/tcache/fastbin 与 exit function 链表目标组合，先确认 glibc、safe-linking 和任意写落点。 |
| [VNCTF2026-ezphp-wp](../raw/pwn/VNCTF2026-ezphp-wp.md) | PHP 扩展 off-by-one 扩大 `vh_edit` 范围后覆盖 `comment` 指针字段；用 `/proc/self/maps` 泄露基址并把 `efree@GOT` 改成 `system`。 |
| [VNCTF2026-real-world-wp](../raw/pwn/VNCTF2026-real-world-wp.md) | 原样 Rsync 3.3.0 服务，先用 CVE-2024-12085 未初始化数据泄漏定位地址，再用 CVE-2024-12084 checksum 堆溢出转任意写。 |
| [VNCTF2026-recode-wp](../raw/pwn/VNCTF2026-recode-wp.md) | Protocol Buffers 对象 UAF 进入 tcache poisoning、任意读写和栈 ROP，先确认对象生命周期和 libc/heap leak。 |

## 合并与拆分结论

- 保留为 family：raw 覆盖多种生命周期 bug 和 allocator primitive，能提供 heap 首轮后的二级分流。
- 不合并进 `heap-houses-unlink-and-tcache.md`：两页边界分别是 primitive 形成和 metadata/bin 技巧落地。
- 不合并进 `heap-fsop-file-structure-attacks.md`：FSOP 是常见落点，但不是所有 heap UAF 的核心。

## 常见误判

- 只按菜单题模板写 exploit，没有确认 glibc 版本和 safe-linking。
- 源码里有防 double free 标志就停止，未检查异常路径、优化和未初始化变量。
- tcache poisoning 只看 fd 可写，没有处理对齐、key 和 size class。
- custom allocator 直接套 glibc 技巧，忽略自定义 metadata 检查。

## 关联页面

- [pwn-first-pass-red-flags-and-protections.md](pwn-first-pass-red-flags-and-protections.md)
- [heap-houses-unlink-and-tcache.md](heap-houses-unlink-and-tcache.md)
- [heap-fsop-file-structure-attacks.md](heap-fsop-file-structure-attacks.md)
- [runtime-protection-and-tls-exploits.md](runtime-protection-and-tls-exploits.md)
- [format-string.md](format-string.md)
- [pwn-tooling.md](pwn-tooling.md)

## 原始资料

- [heap-uaf-tcache-and-custom-allocator.md](../raw/pwn/heap-uaf-tcache-and-custom-allocator.md)
- [WMCTF2025-palusimulator-wp](../raw/pwn/WMCTF2025-palusimulator-wp.md)
- [D3CTF2019-basic-basic-parser-wp](../raw/pwn/D3CTF2019-basic-basic-parser-wp.md)
- [D3CTF2019-new-heap-wp](../raw/pwn/D3CTF2019-new-heap-wp.md)
- [D3CTF2021-deterministic-heap-wp](../raw/pwn/D3CTF2021-deterministic-heap-wp.md)
