---
type: family
tags: [reverse, family, mobile, firmware, kernel, game-engine]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/mobile-firmware-kernel-and-game-re.md
updated: 2026-06-12
---

# Mobile, Firmware, Kernel and Game RE

## 作用边界

本页是平台环境 family，覆盖 macOS/iOS Mach-O、Objective-C/Swift、dyld、固件解包、IoT/RTOS、内核驱动、eBPF、游戏引擎和 CAN/车载协议。它的价值是先判断目标运行在哪个环境、如何加载、怎么调试、哪些格式/权限/架构会影响分析。

与 [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md) 的区别：本页偏 OS、固件、驱动和底层环境；Android/Flutter/Unity/资源类高频入口优先从 Android/游戏 family 进入。

## 首轮路由

| 信号 | 先确认 | 下一跳 |
|---|---|---|
| Mach-O、IPA、Objective-C selector、Swift 符号 | 架构切片、签名/entitlements、selector 字符串、Swift demangle 和 dyld 依赖 | [disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md)、[frida-angr-lldb-and-x64dbg.md](frida-angr-lldb-and-x64dbg.md) |
| 固件镜像、SquashFS/JFFS2/UBI/CPIO、DTB、uImage/zImage | 先解包文件系统、确认架构/端序/基址，再定位 init 脚本、Web handler 或协议服务 | [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md) |
| `.ko`、`.sys`、IOCTL、eBPF、内核模块 | 用户态/内核态边界、handler、map、probe、设备对象和可观测反馈 | [windows-kernel-ioctl-hidden-feedback-maze.md](windows-kernel-ioctl-hidden-feedback-maze.md)、[loader-vm-image-and-kernel-patterns.md](loader-vm-image-and-kernel-patterns.md) |
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

## 关联页面

- [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md)
- [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)
- [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md)
- [windows-kernel-ioctl-hidden-feedback-maze.md](windows-kernel-ioctl-hidden-feedback-maze.md)
- [loader-vm-image-and-kernel-patterns.md](loader-vm-image-and-kernel-patterns.md)
- [signals-and-hardware.md](signals-and-hardware.md)

## 原始资料

- [mobile-firmware-kernel-and-game-re.md](../raw/reverse/mobile-firmware-kernel-and-game-re.md)
