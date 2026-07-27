---
type: technique
tags: [pwn, heap, uaf, tcache, double-free, object-reuse]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/heap-uaf-tcache-and-custom-allocator.md
  - ../raw/pwn/heap-houses-unlink-and-tcache.md
updated: 2026-07-27
---

# UAF Object Reuse and Tcache Poisoning

## 适用场景

对象释放后仍可读写/调用，或 double free/tcache freelist 可控；通过同尺寸复用、类型混淆和 freelist poisoning 把悬空引用转成地址泄露、任意分配或函数指针覆盖。

## 识别信号

- 删除对象后菜单仍保留索引、指针或回调。
- 同尺寸新对象会占用刚释放地址。
- freed chunk 的 next/key 可修改，或存在 double free 绕过。

## 最小证据

- 证明悬空引用在复用前后指向同一地址。
- 确认 allocator size class、tcache count 和 safe-linking 公式。
- 用受控模式验证读取/写入作用于新对象或 freelist。

## 解法骨架

1. 固定分配/释放序列，复用目标 chunk。
2. 先利用 UAF 泄露 heap/libc/key，再计算编码 freelist pointer。
3. Poison tcache 或重解释对象字段，令后续 allocation 落到目标地址。
4. 选择与版本和退出路径匹配的控制流/数据目标并验证。

## 关键变体

- UAF type confusion：同地址换成不同对象布局。
- Double free：重复进入 freelist，构造任意分配。
- Custom allocator：先恢复其 freelist/bitmap，而非套 glibc。

## 常见陷阱

- safe-linking 编码使用了错误 chunk 地址。
- 新分配尺寸不在同一 bin，无法复用。
- 目标 hook 在当前 glibc 已移除或不可触发。

## 关联技巧

- [heap-uaf-tcache-and-custom-allocator.md](heap-uaf-tcache-and-custom-allocator.md)
- [heap-houses-unlink-and-tcache.md](heap-houses-unlink-and-tcache.md)
- [heap-metadata-and-bin-list-corruption.md](heap-metadata-and-bin-list-corruption.md)

## 原始资料

- [heap-uaf-tcache-and-custom-allocator.md](../raw/pwn/heap-uaf-tcache-and-custom-allocator.md)
- [heap-houses-unlink-and-tcache.md](../raw/pwn/heap-houses-unlink-and-tcache.md)
