---
type: family
tags: [reverse, family, mobile, firmware, kernel, game-engine]
skills: [ctf-mobile, ctf-reverse, ctf-hardware-embedded]
raw:
  - ../raw/reverse/mobile-firmware-kernel-and-game-re.md
  - ../raw/reverse/ACTF2026-abyssgate-wp.md
  - ../raw/reverse/Bugku-Berkeley-wp.md
  - ../raw/reverse/HGAME2026-noncesense-wp.md
  - ../raw/reverse/D3CTF2025-d3kernel-wp.md
  - ../raw/reverse/D3CTF2022-d3w0w-wp.md
updated: 2026-07-06
---

# Mobile, Firmware, Kernel and Game RE

## 作用边界

本页是平台环境 family，覆盖 macOS/iOS Mach-O、Objective-C/Swift、dyld、固件解包、IoT/RTOS、内核驱动、eBPF、游戏引擎和 CAN/车载协议。它的价值是先判断目标运行在哪个环境、如何加载、怎么调试、哪些格式/权限/架构会影响分析。

与 [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md) 的区别：本页偏 OS、固件、驱动和底层环境；Android/Flutter/Unity/资源类高频入口优先从 Android/游戏 family 进入。

## 识别信号

- 附件包含 Mach-O/IPA、固件镜像、文件系统 dump、DTB/uImage/zImage、`.ko`、`.sys`、eBPF、IOCTL client、CAN/UDS 流量或游戏引擎底层资源。
- 分析障碍首先是运行环境：签名/entitlements、dyld、端序/架构、加载基址、设备对象、probe/map、驱动接口或协议帧。
- 用户态代码只是 client、loader 或 UI，真实状态机/校验在固件服务、内核模块、驱动、eBPF 或引擎插件中。
- 同一输入跨越文件系统、设备节点、syscall/ioctl、网络帧或引擎资源边界。

## 最小证据

- 先确认平台、架构、端序、加载基址、入口和可运行/可调试方式。
- 固件题要至少解出文件系统或定位 init/服务/handler；驱动题要列出设备名、IOCTL 号、输入结构和返回/副作用。
- eBPF/probe 题要保留 attach 点、map 常量、触发函数和用户态/内核态数据交接。
- 游戏/车载协议题要保存资源/帧结构、状态字段和一个可重放的最小输入输出样本。

## 首轮路由

| 信号 | 先确认 | 下一跳 |
|---|---|---|
| Mach-O、IPA、Objective-C selector、Swift 符号 | 架构切片、签名/entitlements、selector 字符串、Swift demangle 和 dyld 依赖 | [disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md)、[frida-angr-lldb-and-x64dbg.md](frida-angr-lldb-and-x64dbg.md) |
| 固件镜像、SquashFS/JFFS2/UBI/CPIO、DTB、uImage/zImage | 先解包文件系统、确认架构/端序/基址，再定位 init 脚本、Web handler 或协议服务 | [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md) |
| `.ko`、`.sys`、IOCTL、eBPF、内核模块 | 用户态/内核态边界、handler、map、probe、设备对象和可观测反馈 | [windows-kernel-ioctl-hidden-feedback-maze.md](windows-kernel-ioctl-hidden-feedback-maze.md)、[loader-vm-image-and-kernel-patterns.md](loader-vm-image-and-kernel-patterns.md) |
| WoW64 / Heaven's Gate / 模式切换 | 真实校验是否藏在另一执行模式或另一位宽代码中，dump 后按正确架构解析 | [signal-trace-and-packed-anti-analysis.md](signal-trace-and-packed-anti-analysis.md) |
| Unreal/Unity/Lua 游戏引擎 | 资源、脚本、反射元数据、存档和 native 插件谁控制真实校验 | [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md) |
| CAN/UDS/车载协议 | 帧 ID、DLC、seed-key、routine control 和 replay 边界 | [signals-and-hardware.md](signals-and-hardware.md) |

## 合并与拆分结论

- 保留为 family：这些 raw 分支共同问题是运行环境和载体格式，而不是某个单点恢复技巧。
- 不拆 Mach-O/固件/驱动/游戏引擎独立页：当前还缺少足够多直接 WP 案例，先作为平台路由更稳定。
- 与 pwn/kernel 页面保持边界：本页负责理解驱动/内核接口；当目标转为内存破坏和提权时再 pivot 到 pwn。

## 常见误判

- 固件题未先确认文件系统和架构，就直接在随机 ELF 上读伪代码。
- Mach-O/Swift 题忽略 selector、demangle 和 dyld，导致把运行时派发误当普通 C 调用。
- 驱动题只看用户态 client，没有恢复 IOCTL 号、输入结构和内核侧反馈。
- 游戏题只抽资源，不确认脚本/引擎/插件是否会在运行时覆盖静态数据。

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [ACTF2026-abyssgate-wp](../raw/reverse/ACTF2026-abyssgate-wp.md) | 第二阶段 ELF、eBPF tracepoint 和 `abyss.ko` 共同决定 ioctl 参数和 token/tweak 派生，平台边界比单个用户态函数更重要。 |
| [Bugku-Berkeley-wp](../raw/reverse/Bugku-Berkeley-wp.md) | ELF 中的 libbpf-bootstrap、`attach_uprobe` 和 BPF map 表明真实校验在内核侧 instrumentation，不能只信用户态 `check_flag`。 |
| [HGAME2026-noncesense-wp](../raw/reverse/HGAME2026-noncesense-wp.md) | Client/driver 协议题要先枚举设备对象、IOCTL、nonce、token、blob 生命周期，再分析 AES/HMAC 或 VM 变换。 |
| [D3CTF2025-d3kernel-wp](../raw/reverse/D3CTF2025-d3kernel-wp.md) | Windows kernel/userland 混合逆向要把 R3/R0 反调试、DeviceIoControl 分发表和驱动内 VM 分层处理。 |
| [D3CTF2022-d3w0w-wp](../raw/reverse/D3CTF2022-d3w0w-wp.md) | WoW64 隐藏 64 位逻辑时，先 dump 或切换到正确位宽解析，再把方向序列约束转成 puzzle 求解。 |
| [SUCTF2026-old_binWP](../raw/reverse/SUCTF2026-old_binWP.md) | 固件先 XOR 解包出 IMG0 容器，再修复损坏 ELF 和 TLS 布局；最后还原网络 challenge 与自定义块校验。 |

## 关联页面

- [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md)
- [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)
- [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md)
- [windows-kernel-ioctl-hidden-feedback-maze.md](windows-kernel-ioctl-hidden-feedback-maze.md)
- [loader-vm-image-and-kernel-patterns.md](loader-vm-image-and-kernel-patterns.md)
- [signal-trace-and-packed-anti-analysis.md](signal-trace-and-packed-anti-analysis.md)
- [signals-and-hardware.md](signals-and-hardware.md)

## 原始资料

- [mobile-firmware-kernel-and-game-re.md](../raw/reverse/mobile-firmware-kernel-and-game-re.md)
- [ACTF2026-abyssgate-wp](../raw/reverse/ACTF2026-abyssgate-wp.md)
- [Bugku-Berkeley-wp](../raw/reverse/Bugku-Berkeley-wp.md)
- [HGAME2026-noncesense-wp](../raw/reverse/HGAME2026-noncesense-wp.md)
- [D3CTF2025-d3kernel-wp](../raw/reverse/D3CTF2025-d3kernel-wp.md)
- [D3CTF2022-d3w0w-wp](../raw/reverse/D3CTF2022-d3w0w-wp.md)
