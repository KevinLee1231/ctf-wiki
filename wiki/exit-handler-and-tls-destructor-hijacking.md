---
type: technique
tags: [pwn, glibc, exit-handler, atexit, tls, pointer-mangling]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/runtime-protection-and-tls-exploits.md
  - ../raw/pwn/HGAME2026-ionostream-wp.md
  - ../raw/pwn/LilacCTF2026-na1vm-wp.md
updated: 2026-07-28
---

# Exit-Handler and TLS-Destructor Hijacking

## 适用场景

已经取得任意读写或受限写，但常规 hook 不可用；进程或线程结束时会遍历 `__exit_funcs`、`atexit`、`__cxa_atexit`、TLS destructor 或 loader 清理链，可把退出元数据改成受控调用。

## 识别信号

- 程序必然调用 `exit()` 或线程正常退出，且无法直接覆盖返回地址/传统 hook。
- 可定位 `__exit_funcs`、`dl_fini`、TLS destructor list 或相关 loader 状态。
- 函数指针经 pointer guard/mangling 保存，不是可直接写入的明文地址。
- 已知合法条目可用于反推出 guard 或验证编码公式。

## 最小证据

- 确认实际终止路径会运行清理链，而不是 `_exit`、`abort` 或被信号直接杀死。
- 固定 libc/ld 版本，核对 entry type、字段布局和 pointer-mangling 公式。
- 取得一个已知明文函数与其编码指针，或直接泄露 pointer guard。

## 解法骨架

1. 从 libc/ld 版本确定退出链对象与遍历逻辑。
2. 泄露合法的 mangled `dl_fini` 等条目，按版本公式恢复 pointer guard。
3. 在可写内存构造函数类型、编码后函数指针、参数和 next/idx 字段。
4. 覆盖退出链头或现有条目，再触发正常 `exit()`/线程退出。
5. 以一次无副作用目标调用验证布局和编码，再切换到最终控制流。

## 关键变体

- `__exit_funcs`/`exit_function_list` 与 `ef_cxa` 条目。
- `atexit`/`__cxa_atexit` 注册项。
- TLS destructor/thread-exit 链。
- CET/shadow stack 阻断返回地址覆盖时的非返回型触发。

## 常见陷阱

- 把 mangled 函数指针当裸地址写入。
- 使用错误 libc/ld 偏移，或忽略 loader 与 libc 的相对基址。
- 构造条目类型与参数字段不匹配。
- 将 FILE/FSOP 模板直接套到退出链；两者的遍历器和前置检查不同。

## 关联技巧

- [glibc-file-structure-and-fsop.md](glibc-file-structure-and-fsop.md)
- [runtime-mitigation-pointer-mangling-and-shadow-stack-bypass.md](runtime-mitigation-pointer-mangling-and-shadow-stack-bypass.md)
- [runtime-protection-and-tls-exploits.md](runtime-protection-and-tls-exploits.md)
- [heap-metadata-and-bin-list-corruption.md](heap-metadata-and-bin-list-corruption.md)

## 原始资料

- [runtime-protection-and-tls-exploits.md](../raw/pwn/runtime-protection-and-tls-exploits.md)
- [HGAME2026-ionostream-wp](../raw/pwn/HGAME2026-ionostream-wp.md)
- [LilacCTF2026-na1vm-wp](../raw/pwn/LilacCTF2026-na1vm-wp.md)
