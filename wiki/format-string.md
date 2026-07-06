---
type: technique
tags: [pwn, technique, format-string, leak, arbitrary-write]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/format-string.md
updated: 2026-06-12
---

# Format String Exploitation

## 适用场景

当用户输入被当作 `printf`/`fprintf`/`sprintf`/`scanf` 风格格式串解释，并且可通过 `%p/%s/%n` 等 specifier 泄露或写内存时，优先使用本页。它是具体 technique，不是 Pwn 总入口。

如果只能控制普通栈溢出而没有格式串解释，转 [overflow-basics.md](overflow-basics.md)。如果格式串只提供有限写，后续落点可能转 [stack-pivots-srop-and-seccomp-rop.md](stack-pivots-srop-and-seccomp-rop.md)、[runtime-protection-and-tls-exploits.md](runtime-protection-and-tls-exploits.md) 或 heap 页面。

## 识别信号

- 输入中 `%p`、`%x`、`%s`、`%n` 会影响输出、崩溃或写入计数。
- 栈上能看到用户输入、返回地址、canary、libc/PIE 指针或可控地址。
- 过滤器禁止 `%n`、位置参数、数字或某些字符，但格式串经过 ROT13、编码、截断或二次处理后仍会恢复。
- 可写目标包括 GOT、`__free_hook`、`.fini_array`、返回地址、BSS 状态变量、动态符号表或 printf 内部表。

## 最小证据

- 确认格式串 offset：用户输入在第几个参数，地址跟在 payload 后时如何被读取。
- 至少获得一个稳定 leak：栈地址、canary、PIE、libc 或 GOT 内容。
- 对写入路径，确认 `%n/%hn/%hhn` 是否可用、当前已输出字节数、写入目标是否可写。
- 确认远程 libc、换行/缓冲、输出截断和过滤器行为。

## 解法骨架

1. 用 `%p`/marker 定位栈参数 offset。
2. 泄露 canary/PIE/libc 或验证可写目标。
3. 选择写入粒度：优先 `%hhn/%hn` 分段写，避免一次打印过多字节。
4. 选择落点：GOT/hook、`.fini_array` 循环、返回地址、状态变量或二阶段 ROP。
5. 将 payload 脚本化，处理输出粘连、缓冲和远程地址差异。

## 格式串利用分支

| 场景 | 处理重点 |
|---|---|
| 基础 leak/write | 先稳定 offset，再把 `%p/%s` 泄露和 `%n/%hn/%hhn` 写入分开验证。 |
| 非位置参数或 argument retargeting | 不能用 `%N$` 时，利用顺序消费、栈上指针或 payload 后置地址重定向写目标。 |
| Blind pwn | 无 binary 时先用 leak 建立栈/ELF/libc 轮廓，再逐步定位 GOT、返回地址和 libc 基址。 |
| 过滤或输入变换 | 在本地复现 ROT13、长度截断、大小写转换或黑名单顺序，构造最终进入 printf 的真实格式串。 |
| Canary/PIE leak 后接栈溢出 | 格式串只负责泄露，最终控制流仍由 overflow/ROP 完成。 |
| Hook/GOT/`.fini_array`/dynsym patch | 写入目标取决于 RELRO、glibc 版本和能否多轮触发；`.fini_array` 常用于循环回主函数做多阶段。 |
| 游戏状态或业务变量写 | 不必追求 shell，能直接把 chips/score/flag gate 变量写成目标值时优先短路径。 |

## 常见陷阱

- offset 在本地和远程因环境变量、argv 或 banner 不一致而变化。
- 一次 `%n` 打印大量空格导致超时或触发输出限制。
- 忘记 payload 本身已经输出的字节数，分段写偏移错误。
- RELRO/full RELRO 下继续写 GOT，没换 `.fini_array`、栈或 hook/handler。
- `%s` 读不可映射地址直接崩溃，没有先用 `%p` 验证指针。

## 关联技巧

- [overflow-basics.md](overflow-basics.md)
- [stack-pivots-srop-and-seccomp-rop.md](stack-pivots-srop-and-seccomp-rop.md)
- [ret2csu-dynelf-and-shellcode.md](ret2csu-dynelf-and-shellcode.md)
- [runtime-protection-and-tls-exploits.md](runtime-protection-and-tls-exploits.md)
- [heap-fsop-file-structure-attacks.md](heap-fsop-file-structure-attacks.md)
- [pwn-tooling.md](pwn-tooling.md)

## 原始资料

- [format-string.md](../raw/pwn/format-string.md)
