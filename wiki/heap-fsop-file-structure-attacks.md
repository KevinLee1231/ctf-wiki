---
type: family
tags: [pwn, family, heap, fsop, file-structure, glibc]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/heap-fsop-file-structure-attacks.md
updated: 2026-07-27
---

# Heap FSOP and FILE Structure Attacks

## 作用边界

本页是 glibc FILE 结构、stdout/stdin 劫持、FSOP、IO buffer 和 FILE 相关 heap primitive 的二级 family。它从 heap primitive 或 runtime primitive 进入，负责判断是否应该把泄露/写入落到 `_IO_FILE`、vtable、buffer 指针、`stdin/stdout` 字段或 IO 链表。

## 识别信号

- 程序输出受限、stdout/stdin 可被堆写影响，或 exploit 需要先制造 libc/PIE leak。
- 漏洞点能改 `_IO_2_1_stdout_`、`_IO_2_1_stdin_`、`_IO_buf_base`、`_IO_buf_end`、vtable、fileno、flags 或 wide data。
- glibc 版本在 2.24+ 时有 IO vtable validation，需要考虑绕过或改非 vtable 字段。
- 堆 primitive 来自 fastbin/tcache/unsorted bin、`realloc(ptr,0)`、引用计数溢出或相邻对象覆盖。

## 最小证据

- glibc 版本、FILE 结构布局和目标字段 offset。
- 能写入的粒度：null byte、partial overwrite、full AAW、unsorted bin write、overlap write。
- 输出/输入通道是否可观察，是否能触发 flush、read、puts、exit 或其他 IO 路径。
- 已决定目标是 leak、任意读写、控制流劫持还是 ORW 辅助。

## 路由表

| 证据 | 先验证 | 下一跳 |
|---|---|---|
| stdout leak | flags、write_base/write_ptr、buf_base 是否可改 | 伪造 stdout 泄露 libc/PIE |
| stdin hijack | `_IO_buf_base/end`、read buffer 和 fileno 是否可控 | 改输入缓冲实现任意写或栈写 |
| vtable hijack | glibc 是否做 vtable validation | 2.24+ 需找合法 vtable 区或转非 vtable 字段 |
| unsorted bin 写 stdin/stdout | 写入值是否能落到 IO 字段并触发 | 控制 `_IO_buf_end` 或 mp 结构 |
| `realloc(ptr,0)` | 目标 libc 是否把它当 free，后续指针是否继续使用 | 转 UAF/tcache 路线 |
| refcount wrap | 单字节计数溢出导致对象提前释放 | 先稳定生命周期，再进入 heap UAF |
| stdout 被关闭/不可打印 | 是否能改 fileno 或重新打开输出通道 | 关联 [format-string.md](format-string.md) |

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| FILE/wide-data/exit/TLS 结构提供泄露或控制流 | [file-structure-and-exit-handler-control-flow.md](file-structure-and-exit-handler-control-flow.md) |
| chunk/bin metadata 先提供可写目标或地址泄露 | [heap-metadata-and-bin-list-corruption.md](heap-metadata-and-bin-list-corruption.md) |
| 格式化字符串可先覆盖 FILE 指针/字段或获取地址 | [format-string.md](format-string.md) |

## 合并与拆分结论

- 保留为 family：FILE 结构既可做 leak，也可做输入劫持、vtable 劫持和 ORW 辅助，首轮需要按字段和 glibc 版本分流。
- 不合并进 `heap-uaf-tcache-and-custom-allocator.md`：heap 页负责 primitive 形成，本页负责 FILE 结构落点。
- 不合并进 `runtime-protection-and-tls-exploits.md`：该页覆盖更宽的 runtime callback/protection，FSOP 需要独立的 glibc IO 结构判断。

## 常见误判

- 套旧版 FSOP payload，没有检查 glibc 2.24+ vtable validation。
- 改了 FILE 字段但没有触发对应 IO 路径，导致无输出或不崩不漏。
- 只看 stdout，不检查 stdin buffer 是否能作为更稳定的写入口。
- Partial overwrite 没考虑 FILE 字段对齐和已有高字节。

## 关联页面

- [heap-uaf-tcache-and-custom-allocator.md](heap-uaf-tcache-and-custom-allocator.md)
- [heap-houses-unlink-and-tcache.md](heap-houses-unlink-and-tcache.md)
- [runtime-protection-and-tls-exploits.md](runtime-protection-and-tls-exploits.md)
- [format-string.md](format-string.md)
- [ret2csu-dynelf-and-shellcode.md](ret2csu-dynelf-and-shellcode.md)
- [pwn-tooling.md](pwn-tooling.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [D3CTF2019-ezfile-wp](../raw/pwn/D3CTF2019-ezfile-wp.md) | tcache double free 配合 FILE fileno 和受限 ORW，先复用已有 FD 而不是追求 shell。 |
| [HGAME2026-diary-keeper-wp](../raw/pwn/HGAME2026-diary-keeper-wp.md) | glibc 2.35 off-by-null 伪造合并并泄露 libc/heap，后续不是普通 UAF，而是 `_IO_list_all` + house of obstack。 |
| [RCTF2025-mstr-wp](../raw/pwn/RCTF2025-mstr-wp.md) | 运行时字符串对象重叠形成越界读写并最终打 FSOP，先确认对象布局、libc 和 FILE 结构目标。 |
| [RCTF2025-rd-wp](../raw/pwn/RCTF2025-rd-wp.md) | 未初始化 task 指针可踩 stdout，先泄露 heap/libc，再构造 fake stdout / fake FILE 控制执行流。 |
| [0xGame2023-week3-shellcode-but-FOP-wp](../raw/pwn/0xGame2023-week3-shellcode-but-FOP-wp.md) | 本题需要同时解决随机入口、禁用 execve、随机文件名和关闭 stdout 四个约束。第一阶段利用调用前寄存器状态做二次读取，第二阶段通过目录枚举定位文件，再使用 ORW 读取；这比假设文件名固定或直接套 `shellcraft.sh()` 更符合实际沙箱边界。 |
| [ACTF2023-qemu-playground-2-wp](../raw/pwn/ACTF2023-qemu-playground-2-wp.md) | 本题的利用链为：逆向第一阶段口令、通过认证开放 `ptr`、利用 MMIO 末端 4 字节越界改写指针低位、通过 PIO 得到同一 4 GiB 窗口内的读写、沿真实对象指针泄露 libc，最后用宽字符 FSOP 栈迁移到 Host ROP。 |
| [UMDCTF2025-aura-wp](../raw/pwn/UMDCTF2025-aura-wp.md) | 先破坏仍会被使用的 `FILE` 对象，再借 `fread` 完成受控写入。审计时应检查 `FILE *` 是否被误交给 `read`、`memcpy` 等裸内存接口，以及后续标准 I/O 能否把伪造的缓冲区边界、fd 和状态位转化为任意读写。 |

## 原始资料

- [heap-fsop-file-structure-attacks.md](../raw/pwn/heap-fsop-file-structure-attacks.md)
