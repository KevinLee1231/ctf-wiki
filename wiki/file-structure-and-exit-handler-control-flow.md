---
type: technique
tags: [pwn, fsop, file-structure, atexit, tls]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/heap-fsop-file-structure-attacks.md
  - ../raw/pwn/runtime-protection-and-tls-exploits.md
updated: 2026-07-27
---

# FILE Structure and Exit-Handler Control Flow

## 适用场景

任意/受限写可影响 glibc `FILE`、I/O buffer、wide-data、TLS destructor 或 `atexit` 元数据；程序的 flush、close、exit 或线程退出路径提供稳定触发点。

## 识别信号

- 可泄露/覆盖 `_IO_2_1_stdout_`、`stdin`、`FILE` 指针或退出链。
- 常规 malloc hook 不可用，但程序必然调用 `exit`、flush 或 close。
- pointer mangling、vtable check 和 wide-data 布局与 libc 版本相关。

## 最小证据

- 固定 libc/loader，取得真实结构偏移和校验逻辑。
- 确认写原语能覆盖所需字段与指针编码。
- 证明目标触发路径确实会遍历伪造结构。

## 解法骨架

1. 选择最少字段的泄露或控制流 primitive。
2. 在可写内存构造 FILE/wide-data/exit entry，并满足 flags、mode、lock、vtable 等检查。
3. 必要时恢复/绕过 pointer guard，再链接可控函数与参数。
4. 触发 flush/exit 并在目标版本下端到端验证。

## 关键变体

- FILE leak/buffer redirection。
- Wide-data/FSOP 控制流。
- `atexit`/TLS destructor 链与指针加密。

## 常见陷阱

- 复制其它 libc 版本结构偏移。
- 忽略 lock、mode、vtable range 等前置检查。
- 程序通过 `_exit` 结束，根本不会触发清理链。

## 关联技巧

- [heap-fsop-file-structure-attacks.md](heap-fsop-file-structure-attacks.md)
- [runtime-protection-and-tls-exploits.md](runtime-protection-and-tls-exploits.md)
- [runtime-mitigation-pointer-mangling-and-shadow-stack-bypass.md](runtime-mitigation-pointer-mangling-and-shadow-stack-bypass.md)

## 原始资料

- [heap-fsop-file-structure-attacks.md](../raw/pwn/heap-fsop-file-structure-attacks.md)
- [runtime-protection-and-tls-exploits.md](../raw/pwn/runtime-protection-and-tls-exploits.md)
