---
type: technique
tags: [pwn, glibc, file-structure, fsop, wide-data]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/heap-fsop-file-structure-attacks.md
  - ../raw/pwn/HGAME2026-diary-keeper-wp.md
updated: 2026-07-28
---

# glibc FILE Structure and FSOP

## 适用场景

堆或任意写原语可影响 glibc `FILE`、I/O buffer、wide-data、codecvt 或 `_IO_list_all`；程序的输出、flush、close 或正常退出会遍历这些对象，从而提供泄露、任意读写或控制流触发。

## 识别信号

- 可泄露或覆盖 `_IO_2_1_stdout_`、`stdin`、`FILE *` 或 `_IO_list_all`。
- 高版本 glibc 无传统 malloc hook，但可构造 fake stream/wide-data。
- 程序会调用 `puts/printf/fwrite/fflush/fclose/exit`，且相应 stream 可达。
- libc 版本的 vtable check、wide vtable、lock 和 mode 字段决定可用链。

## 最小证据

- 固定目标 libc，取得实际 `FILE`/wide-data 字段偏移和 vtable 校验逻辑。
- 确认写原语覆盖范围、对齐、NUL 限制以及触发时 stream 状态。
- 在同版本本地环境中证明目标 I/O 路径会遍历伪造字段。

## 解法骨架

1. 先选目标 primitive：stdout leak、buffer redirection、任意写或 FSOP 控制流。
2. 从目标 I/O 函数反推最小必要字段，而不是复制完整模板。
3. 在可写内存布置 `FILE`、wide-data、lock、buffer 和 vtable 相关对象。
4. 满足 flags、mode、buffer range 与 vtable range 检查后触发 flush/close。
5. 对泄露和控制流分别做正向验证，确认没有依赖偶然堆布局。

## 关键变体

- `FILE` buffer/指针重定向形成泄露或任意读写。
- `_IO_list_all` 与伪造 stream 链。
- wide-data/wide-vtable 路线。
- house of obstack 等堆结构最终转入 I/O 触发。

## 常见陷阱

- 复制其它 glibc 版本的结构偏移和现成 payload。
- 忽略 `_lock`、`_mode`、buffer 大小关系或 vtable 范围检查。
- 混入 `__exit_funcs`/TLS destructor；它们不是 FILE 对象，见独立退出链页面。
- 程序通过 `_exit` 或异常终止，根本不会执行预期 flush。

## 关联技巧

- [exit-handler-and-tls-destructor-hijacking.md](exit-handler-and-tls-destructor-hijacking.md)
- [heap-fsop-file-structure-attacks.md](heap-fsop-file-structure-attacks.md)
- [heap-metadata-and-bin-list-corruption.md](heap-metadata-and-bin-list-corruption.md)
- [uaf-object-reuse-and-tcache-poisoning.md](uaf-object-reuse-and-tcache-poisoning.md)

## 原始资料

- [heap-fsop-file-structure-attacks.md](../raw/pwn/heap-fsop-file-structure-attacks.md)
- [HGAME2026-diary-keeper-wp](../raw/pwn/HGAME2026-diary-keeper-wp.md)
