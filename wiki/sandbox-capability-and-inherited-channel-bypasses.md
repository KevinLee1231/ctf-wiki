---
type: technique
tags: [pwn, sandbox, seccomp, capabilities, file-descriptor, technique]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/ACTF2023-master-of-orw-wp.md
  - ../raw/pwn/ACTF2023-young-man-escape-wp.md
  - ../raw/pwn/ACTF2025-AFL-sandbox-wp.md
  - ../raw/pwn/UMDCTF2022-tracestory-wp.md
  - ../raw/pwn/0xGame2025-week4-Jail大逃亡-wp.md
  - ../raw/pwn/WMCTF2024-test-your-nc-3-wp.md
updated: 2026-07-27
---

# Sandbox Capability and Inherited-Channel Bypasses

## 适用场景

受限进程的 syscall、路径、语言对象或输出接口看似被封锁，但启动前遗留的 capability、兄弟进程、文件描述符、共享内存、挂载能力、父进程协议或可写加载路径仍能表达同等能力。攻击重点是枚举真实能力图，而不是只绕过滤器字符串。

## 识别信号

- seccomp 使用 denylist，仍允许 io_uring、新 mount API、`ptrace` 或其它间接系统调用。
- 进程保留 `CAP_SYS_ADMIN`、可访问 `/proc/1/root`，或与未受过滤进程共享同一内核对象。
- 高编号 fd、pipe、socketpair、forkserver、共享内存或父进程 IPC 被继承。
- 受限代码能写入高权限解释器后续加载的模块、配置或工作目录。
- 文件已经在降权/删除/权限收紧前打开，并通过 `pass_fds` 等机制传入。

## 最小证据

- 导出实际 seccomp BPF、capability、namespace、mount、进程树和 fd 表。
- 标注每项能力的创建时刻、持有者、读写方向和父子/兄弟关系。
- 证明一个被允许的原语能替代被禁止操作，例如 SQE 表达文件 I/O。
- 明确最终数据通道或执行落点，不把“可通信”误认为“已读 flag”。

## 解法骨架

1. 从运行器和启动顺序重建进程、权限、namespace、filter 安装和 fd 继承时间线。
2. 枚举 syscall 白名单以外的 capability：io_uring、ptrace、mount API、共享内存、父进程协议、可写 import path。
3. 选择最短能力转换链，把可用原语变成文件读、跨进程写、父进程反序列化或高权限加载。
4. 按真实协议处理队列、长度前缀、异步完成和阻塞语义。
5. 在隔离环境逐段证明数据流，最后再压缩 payload 或远程化。

## 关键变体

- io_uring 将 open/read/write 编码为 SQE，过滤传统 syscall 不会自动过滤 ring 操作。
- 新 mount API 配合 `CAP_SYS_ADMIN` 可绕过只拦截旧 `mount` 的 denylist。
- `ptrace` 可改写过滤器安装前 fork 出的兄弟进程，使其代执行禁用 syscall。
- 已打开 fd 的访问权不因路径删除或权限变化失效。
- forkserver/Queue 等父进程协议既可作为输出信道，也可能触发高权限反序列化。

## 常见陷阱

- 只读 seccomp 规则，没有检查 capability、namespace、fd 和其它进程。
- 把 chroot、seccomp、capability 和 namespace 当成同一隔离层。
- 猜测高编号 fd 用途，不从启动器或 `/proc/self/fd` 验证协议方向。
- 能写文件就直接宣称提权，没有证明高权限进程何时、以何种路径加载它。
- 忽略 IPC 长度前缀、异步队列和阻塞行为，导致本地偶发、远程卡死。

## 关联技巧

- [python-vm-and-proc-sandbox-escape.md](python-vm-and-proc-sandbox-escape.md)
- [seccomp-ret2dlresolve-and-runtime-primitives.md](seccomp-ret2dlresolve-and-runtime-primitives.md)
- [pyjails.md](pyjails.md)
- [source-backdoors-and-restricted-shell-tricks.md](source-backdoors-and-restricted-shell-tricks.md)
- [pwn-tooling.md](pwn-tooling.md)

## 原始资料

- [ACTF2023-master-of-orw-wp.md](../raw/pwn/ACTF2023-master-of-orw-wp.md)
- [ACTF2023-young-man-escape-wp.md](../raw/pwn/ACTF2023-young-man-escape-wp.md)
- [ACTF2025-AFL-sandbox-wp.md](../raw/pwn/ACTF2025-AFL-sandbox-wp.md)
- [UMDCTF2022-tracestory-wp.md](../raw/pwn/UMDCTF2022-tracestory-wp.md)
- [0xGame2025-week4-Jail大逃亡-wp.md](../raw/pwn/0xGame2025-week4-Jail大逃亡-wp.md)
- [WMCTF2024-test-your-nc-3-wp.md](../raw/pwn/WMCTF2024-test-your-nc-3-wp.md)
