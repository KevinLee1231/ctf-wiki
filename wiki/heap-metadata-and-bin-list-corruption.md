---
type: technique
tags: [pwn, heap, unlink, largebin, unsorted-bin, metadata]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/heap-houses-unlink-and-tcache.md
  - ../raw/pwn/heap-fsop-file-structure-attacks.md
updated: 2026-07-28
---

# Heap Metadata and Bin-List Corruption

## 适用场景

堆溢出、off-by-one 或任意写可破坏 chunk size、prev_size、fd/bk 或 largebin/unsorted-bin 链表，从分配器一致性操作中导出受控写、重叠块或地址泄露。

## 识别信号

- 可覆盖相邻 chunk header 或 free chunk 链接字段。
- glibc 版本和分配序列允许 unlink、largebin 或 unsorted-bin 路线。
- 目标写地址可通过链表指针、size class 或 consolidate 影响。

## 最小证据

- 固定 libc/allocator 版本并画出每次操作后的 chunk 布局。
- 逐条列出 allocator integrity check 及满足方式。
- 在调试器中证明 primitive 是可控写/重叠，而非仅崩溃。

## 解法骨架

1. 通过最短 grooming 建立稳定相邻关系。
2. 只修改获得 primitive 所需的 metadata，保留其它一致性字段。
3. 触发 free/malloc/consolidate/bin insertion 获得重叠或写原语。
4. 再转向 hook、FILE、退出处理器或对象指针等版本适配目标。

## 关键变体

- Unsafe unlink 与 fake chunk。
- Unsorted-bin 泄露/写。
- Largebin attack 与 size-sorted list 操纵。

## 常见陷阱

- 按旧 glibc 模板套用，未核对新版本检查。
- Grooming 依赖未显式记录，远端分配顺序改变即失败。
- primitive 只有受限值/受限地址，却按任意写使用。

## 关联技巧

- [heap-houses-unlink-and-tcache.md](heap-houses-unlink-and-tcache.md)
- [heap-uaf-tcache-and-custom-allocator.md](heap-uaf-tcache-and-custom-allocator.md)
- [glibc-file-structure-and-fsop.md](glibc-file-structure-and-fsop.md)
- [exit-handler-and-tls-destructor-hijacking.md](exit-handler-and-tls-destructor-hijacking.md)

## 原始资料

- [heap-houses-unlink-and-tcache.md](../raw/pwn/heap-houses-unlink-and-tcache.md)
- [heap-fsop-file-structure-attacks.md](../raw/pwn/heap-fsop-file-structure-attacks.md)
