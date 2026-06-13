---
type: family
tags: [pwn, family, sandbox, vm, proc, emulator]
skills: [ctf-pwn, ctf-misc, ctf-reverse]
raw:
  - ../raw/pwn/python-vm-and-proc-sandbox-escape.md
updated: 2026-06-12
---

# Python, VM and /proc Sandbox Escape

## 作用边界

本页是 Pwn 视角下的 sandbox、解释器、VM、`/proc` 和运行时文件系统逃逸 family。它处理的问题不是普通 Python jail，也不是单纯 VM 逆向，而是受限执行环境中如何把可用 primitive 扩展成文件读写、内存写入、syscall、宿主执行或检查绕过。

如果主要障碍是 Python 对象链和命名空间过滤，优先看 [pyjails.md](pyjails.md)。如果主要任务是还原复杂字节码语义，优先看 Reverse VM family；只有当已经出现低级读写/执行 primitive 或 OS 边界逃逸时进入本页。

## 识别信号

- 题目给出 Python/Busybox/Lua/自定义 VM/emulator/FUSE/CUSE，且运行在 seccomp、chroot、容器或受限 shell 中。
- 可访问 `/proc/self/mem`、`/proc/*/maps`、pipe、fifo、FUSE/CUSE 设备、`process_vm_readv`、文件大小检查或特殊 syscall。
- VM 指令、emulator opcode 或 syscall blacklist 可以被非常规指令、内部 helper 或寄存器残留绕过。
- 成功路线更像“从沙箱内构造读写/执行通道”，而不是传统 ret2libc。

## 最小证据

- 已确认边界：语言 sandbox、系统调用过滤、文件系统限制、设备回调、VM loader 还是 emulator。
- 能列出可用 primitive：任意文件读、任意写、内存读写、可控 opcode、可执行页、pipe/fifo 或 syscall 子集。
- 能复现一个边界差异：检查层和使用层看到的文件/内存/指令不同。
- 有最终落点：写 `/proc/self/mem`、绕文件大小、触发内部 syscall、逃出 restricted shell 或拿到宿主文件。

## 路由表

| 证据 | 先验证 | 下一跳 |
|---|---|---|
| Python sandbox 但目标是对象链 | 可用 builtins、class hierarchy、import 和字符串构造 | [pyjails.md](pyjails.md) |
| `/proc/self/mem` 可写 | maps 中目标地址、权限和写入 offset 是否稳定 | patch GOT/code/object 或写 shellcode |
| `process_vm_readv` / `/proc` 失败反馈 | 失败状态是否泄露权限、地址或 sandbox 边界 | 转文件/内存 oracle |
| FIFO/pipe 文件大小绕过 | check 和 use 是否针对同一路径/同一 fd | `mkfifo` 或 named pipe 绕固定大小检查 |
| FUSE/CUSE/设备回调 | 内核或服务端会回调用户态文件系统 | 转 [linux-kernel-exploit-basics.md](linux-kernel-exploit-basics.md) 或 race primitive |
| VM/emulator opcode 过滤 | 过滤发生在文本汇编、loader 还是解释器执行阶段 | [oob-jit-parser-primitives-family.md](oob-jit-parser-primitives-family.md) |
| syscall blacklist | 是否可用 `sysenter`、vDSO、uncommon syscall 或内部 helper | [seccomp-ret2dlresolve-and-runtime-primitives.md](seccomp-ret2dlresolve-and-runtime-primitives.md) |
| restricted shell / Busybox | 命令、重定向、通配符、环境变量和 applet 是否可用 | [source-backdoors-and-restricted-shell-tricks.md](source-backdoors-and-restricted-shell-tricks.md) |

## 合并与拆分结论

- 保留为 family：raw 覆盖 Python、VM、FUSE/CUSE、`/proc`、fifo、emulator 和 shell trick，靠单一 technique 无法概括。
- 不合并进 `pyjails.md`：本页更偏 OS/VM/pwn primitive，`pyjails.md` 偏语言对象链。
- 不合并进 `seccomp-ret2dlresolve-and-runtime-primitives.md`：该页处理 syscall/ROP 落地，本页处理沙箱边界如何先变成 primitive。

## 常见误判

- 看到 Python 就直接按 pyjail 解，忽略 `/proc`、fifo 或文件系统使用差异。
- 只绕过 opcode 文本过滤，没有确认解释器内部实际执行的 opcode。
- `mkfifo` 绕过大小检查时没处理阻塞读写，导致本地可行远程卡住。
- 使用 `/proc/self/mem` 前没固定映射地址和权限，写到不可执行/错误段。

## 关联页面

- [pyjails.md](pyjails.md)
- [vm-z3-sandbox-and-game-basics.md](vm-z3-sandbox-and-game-basics.md)
- [oob-jit-parser-primitives-family.md](oob-jit-parser-primitives-family.md)
- [seccomp-ret2dlresolve-and-runtime-primitives.md](seccomp-ret2dlresolve-and-runtime-primitives.md)
- [linux-kernel-exploit-basics.md](linux-kernel-exploit-basics.md)
- [source-backdoors-and-restricted-shell-tricks.md](source-backdoors-and-restricted-shell-tricks.md)
- [pwn-tooling.md](pwn-tooling.md)

## 原始资料

- [python-vm-and-proc-sandbox-escape.md](../raw/pwn/python-vm-and-proc-sandbox-escape.md)
