---
type: technique
tags: [pwn, kernel, kaslr, kpti, smep, smap, technique]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/kaslr-kpti-smep-and-kernel-debugging.md
updated: 2026-06-11
---

# Kernel KASLR, KPTI, SMEP and SMAP Bypass

## 适用场景

Linux kernel pwn 已经有可触发 bug 或低级原语，但 exploit 卡在内核保护组合上：KASLR/FGKASLR 隐藏地址、KPTI 要求安全返回用户态、SMEP/SMAP 阻止直接执行或访问用户态 payload，题目环境又常常缺少完整符号、需要在 QEMU/initramfs 中调试。

本页不是 kernel 工具清单，而是用于回答：当前 primitive 能绕过哪些保护、还缺哪类 leak 或 write target、最终应该走 kernel ROP、`modprobe_path`、`core_pattern`、ret2usr 还是其它路线。

## 识别信号

- 附件包含 kernel image、initramfs、启动脚本、内核模块、字符设备、ioctl 接口或 `/dev/*` 交互点。
- 启动参数或 dmesg 暗示 `kaslr`、KPTI、SMEP/SMAP、FGKASLR、`CONFIG_KALLSYMS` 限制、module load address 随机化。
- 已有 bug 能造成 UAF、OOB、任意读写、栈溢出、函数指针覆盖或 cred 相关对象污染，但不知道如何转成 root shell。
- 本地调试和远程环境差异明显，需要通过 QEMU、GDB、initramfs、virtio-9p 或符号偏移重建稳定 exploit。

## 最小证据

- 记录启动脚本、kernel command line、CPU 保护位、内核版本、模块加载方式和 initramfs 内容。
- 明确当前 primitive：能 leak 什么地址、能写什么地址、能否控制 RIP、能否重复触发、触发后是否还能返回用户态。
- 至少有一个可验证基址或目标：kernel text、module base、heap object、栈 leak、`modprobe_path`、`core_pattern`、cred/task_struct 或可调用 gadget。
- 已确认 KPTI 返回路径：是否可用 `swapgs_restore_regs_and_return_to_usermode` / signal handler / trampoline，或者题目是否关闭了相关保护。

## 解法骨架

1. 先复现 bug，并把 crash 降成 primitive 表：leak、AAW、控制函数指针、栈 pivot、UAF object overlap。
2. 处理地址随机化：用栈/堆/object leak 恢复 kernel base；FGKASLR 场景优先寻找未随机化区域、固定 trampoline 或数据指针。
3. 根据保护选择目标：
   - 有稳定 AAW：优先评估 `modprobe_path`、`core_pattern`、cred 指针或函数表。
   - 有 RIP 控制且 SMEP 开：构造 kernel ROP，不跳用户态 shellcode。
   - SMEP/SMAP 关或可控 CR4：可考虑 ret2usr，但要确认题目环境允许。
   - KPTI 开：kernel ROP 结束后必须走正确 trampoline 返回用户态。
4. 调试阶段先在 QEMU 里加符号、断点和共享目录；远程化前移除调试依赖，只保留可自动投递 exploit。
5. 最终 proof 以 `commit_creds(prepare_kernel_cred(0))`、覆盖执行路径、helper 触发或 root shell 的稳定输出为准。

## 关键变体

| 变体 | 判断与路线 |
|---|---|
| KASLR stack / heap leak | 如果能泄露内核栈、函数指针或对象地址，先用固定偏移恢复 kernel base，再决定 ROP 或数据覆盖目标。 |
| FGKASLR | 不要假设所有 text 偏移稳定；优先找未随机化区、trampoline、导出符号附近 gadget 或数据结构目标。 |
| KPTI trampoline 返回 | kernel ROP 拿到 root 后，要保存用户态 `cs/ss/rflags/rsp/rip`，再通过 trampoline 或等价路径回用户态。 |
| SMEP/SMAP | SMEP 阻止执行用户页，SMAP 阻止访问用户页；路线通常转为 kernel ROP、内核内存 payload 或 CR4/保护位绕过。 |
| `modprobe_path` / `core_pattern` | AAW 足够稳定时可覆盖 helper 路径，再触发未知格式二进制或 core dump，避免复杂 ROP。 |
| 无完整 kallsyms | 通过 GDB 断点、已知函数调用、`/proc/kallsyms` 可见符号、module base 或运行时寄存器推导目标偏移。 |
| initramfs / virtio-9p 调试 | 解包 initramfs、改 `/init`、加共享目录和 GDB symbol，是把本地 exploit 调稳的关键工程步骤。 |
| exploit delivery | 远程环境常限制文件大小和交互方式；静态编译、strip、压缩/base64、分段写入都属于 exploit 稳定性的一部分。 |

## 常见陷阱

- 只看到 KASLR 就盲找 gadget，没有先证明当前 primitive 能提供哪个基址 leak。
- SMEP 开启时仍尝试跳用户态 shellcode，或者 KPTI 开启后直接 `iretq` 导致崩溃。
- 本地 QEMU 修改了 initramfs 或启动参数，却把这些调试条件误当成远程条件。
- 只依赖 `/proc/kallsyms`，没有准备符号不可见时的偏移恢复路线。
- 覆盖 `modprobe_path` / `core_pattern` 后没有确认触发条件，导致写对地址但 helper 不执行。

## 关联技巧

- [cross-primitive-escape-and-hybrid-exploit-map.md](cross-primitive-escape-and-hybrid-exploit-map.md)
- [linux-kernel-exploit-basics.md](linux-kernel-exploit-basics.md)
- [kernel-uaf-race-and-slab-techniques.md](kernel-uaf-race-and-slab-techniques.md)
- [seccomp-ret2dlresolve-and-runtime-primitives.md](seccomp-ret2dlresolve-and-runtime-primitives.md)
- [pwn-first-pass-red-flags-and-protections.md](pwn-first-pass-red-flags-and-protections.md)
- [pwn-tooling.md](pwn-tooling.md)

## 原始资料

- [kaslr-kpti-smep-and-kernel-debugging.md](../raw/pwn/kaslr-kpti-smep-and-kernel-debugging.md)
