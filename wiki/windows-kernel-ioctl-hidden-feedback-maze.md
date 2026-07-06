---
type: technique
tags: [reverse, technique, windows-kernel, driver, ioctl, maze, side-channel]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/Spirit2026-5-kernelmaze-wp.md
updated: 2026-05-21
---

# Windows Kernel Driver IOCTL Hidden Feedback Maze

## 适用场景

Windows 驱动题同时给出 `.sys` 和用户态客户端，客户端只暴露 `DeviceIoControl` 输入，不直接回显移动结果、状态变化或是否撞墙。题目表面像“走迷宫”，实际难点是从驱动内部恢复 IOCTL 分发表、状态机和隐藏反馈通道，再写 replay-from-reset 探图器。

## 识别信号

- 附件包含 `*.sys`、用户态 `*.exe` 和简短 readme，用户态程序只负责打开设备和发送输入。
- `DriverEntry` 创建设备名、符号链接和 `IRP_MJ_DEVICE_CONTROL`，关键逻辑在 IOCTL dispatch。
- 字符串窗口几乎没有明文设备名、Event 名或注册表路径，调用点附近存在“密文拷贝到栈上 + 宽字符 XOR + `RtlInitUnicodeString`”模板。
- `INFO/RESET/STEP/COMMIT` 一类 IOCTL 暗示驱动维护内部状态，移动结果可能通过 Event、注册表、LastError、用户态探针页或 ObCallback 间接泄漏。

## 最小证据

- 已恢复用户态打开路径，例如 `\\.\KernelMaze` 或同类设备符号链接。
- 已定位 `IRP_MJ_DEVICE_CONTROL` dispatch 和 IOCTL 分发表，并能区分初始化、重置、单步移动和最终提交命令。
- 至少找到一个可观测反馈通道，例如 Event 是否 signal、注册表 `LastMove/LastSeq`、`GetLastError()` 特殊值、用户态探针页，或进程对象回调行为。
- 能从 `RESET` 开始重放同一路径并得到稳定结果；否则探图器会被隐藏状态污染。

## 解法骨架

1. 从 `DriverEntry` 入手，恢复运行时解密的设备名、符号链接、注册表路径和 Event 名。
2. 拆 `IRP_MJ_DEVICE_CONTROL` 分发表，给每个 IOCTL 建立语义名：`INFO`、`RESET`、`STEP`、`COMMIT`。
3. 先理解 `RESET`，因为它通常会清空或初始化所有隐藏反馈通道。
4. 对 `STEP` 写最小调用器；每次从 `RESET` 开始重放已知路径，再试探一个未知方向。
5. 对每次试探同时读取多个通道：Event、注册表、LastError、用户态探针页、ObCallback 触发情况。
6. 用 BFS/DFS 建图，记录通路、墙、反馈通道和路径长度；不要只存当前位置。
7. 得到完整图后求最短路，再通过官方客户端或复现后的 `COMMIT` 验证最终 flag。
8. 如果 `COMMIT` 直接调用只返回乱码，检查官方客户端是否参与额外解密、校验或最终呈现。

## 关键变体

- **运行时字符串模板。** 驱动把宽字符密文搬到栈上再按下标 XOR，静态字符串窗口无效时，应从 `RtlInitUnicodeString` 调用点回溯。
- **多通道反馈。** Event 可能只适合 bootstrap；后续需要注册表、LastError 或内存探针补全反馈。
- **最短路约束。** 有些迷宫题不是“到终点即可”，最终 flag 可能绑定最短路径或官方客户端状态。
- **官方客户端后处理。** 自己发 `COMMIT` 得到异常输出时，先确认客户端是否额外处理结果。

## 常见陷阱

- 只看用户态客户端，会误以为题目没有反馈；真正反馈可能由驱动写入系统对象或线程状态。
- 只靠 `DeviceIoControl` 返回 buffer，容易错过 `TEB->LastErrorValue`、注册表或 Event 旁路输出。
- 只用一个全局状态持续走迷宫，难以稳定探边；从 `RESET` 重放到 frontier 更容易复现和回滚。
- 看到 kernel driver 就直接切 pwn；如果目标是理解 IOCTL 状态机和隐藏反馈，应先按 reverse 处理。

## 关联技巧

- [loader-vm-image-and-kernel-patterns.md](loader-vm-image-and-kernel-patterns.md)
- [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md)
- [self-decrypting-strings-and-lattice-patterns.md](self-decrypting-strings-and-lattice-patterns.md)
- [vmp-client-server-smc-rc4-recovery.md](vmp-client-server-smc-rc4-recovery.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [SU_LockWP](../raw/reverse/SU_LockWP.md) | Inno Setup、Rust overlay、锁屏程序和内核驱动多层嵌套；最终 IOCTL 中 XXTEA-like dword 校验在驱动层。 |
| [SU_RevirdWP](../raw/reverse/SU_RevirdWP.md) | 外层魔改 AES 解出第二阶段 EXE，随后通过 `\\.\Revird` 与驱动 op case 协同完成 AES-like 校验。 |
| [VNCTF2026-shadow-wp](../raw/reverse/VNCTF2026-shadow-wp.md) | 用户态迷宫只触发 `Sleep(0x32)`；真实校验在反射加载驱动、PTE hook、键盘记录和基于 `KeDelayExecutionThread` 参数解密的 shellcode。 |

## 原始资料

- [Spirit2026-5-kernelmaze-wp.md](../raw/reverse/Spirit2026-5-kernelmaze-wp.md)
