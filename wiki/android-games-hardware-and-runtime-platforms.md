---
type: family
tags: [reverse, family, android, games, runtime-platforms]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/android-games-hardware-and-runtime-platforms.md
  - ../raw/reverse/WMCTF2025-appfriend-wp.md
  - ../raw/reverse/WMCTF2025-want2become-magicalgirl-wp.md
updated: 2026-07-06
---

# Android, Games, Hardware and Runtime Platforms

## 作用边界

本页是运行时平台 family，负责把 Android/APK、Flutter、Unity/Godot/Roblox、Electron/Node、硬件描述、SGX、AS/400、Glulx 等“平台先于算法”的逆向题分流到具体分析路线。它不再作为单一 technique 使用。

如果题目已经明确是语言运行时对象、Python 字节码、固件/内核驱动或反调试，应从本页转入对应 family / technique。

## 识别信号

- 附件是 APK/IPA、Flutter bundle、Unity/Godot/Roblox/RPG Maker 资源、Electron/Tauri 包、SGX/legacy 平台文件或硬件描述。
- 静态入口只负责 UI、资源加载或桥接调用，真实校验可能在 native `.so`、engine plugin、脚本资源、运行时 patch 或平台 API 中。
- 题目需要先处理平台格式、资源索引、反射元数据、JNI/FFI 边界、hook 点或运行环境，才能讨论算法。
- 同一输入在 UI 层、脚本层、native 层或资源层之间被多次转换。

## 最小证据

- 明确载体格式、运行平台、架构、入口组件和真实校验可能所在层。
- 至少记录一条从用户输入到校验函数/资源/插件/native 调用的数据流。
- 如果依赖动态 hook，要保存 hook 位置、参数、返回值和触发条件，而不是只给最终 patch。
- 如果是游戏/资源题，要能说明资源、脚本、存档和 native 插件谁控制最终状态。

## 首轮路由

| 信号 | 先确认 | 下一跳 |
|---|---|---|
| APK、smali、JNI、`RegisterNatives`、native `.so` | Java/Flutter 层是否只是入口；真实 key、cipher、hook 或 anti-debug 是否在 native 层 | [frida-angr-lldb-and-x64dbg.md](frida-angr-lldb-and-x64dbg.md)、[anti-analysis.md](anti-analysis.md) |
| Android DEX 运行时 patch、`/proc/self/maps`、动态加载 | 静态 DEX 是否被 native 代码改写，需不需要重建 patched DEX | [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md) |
| Unity/Godot/Roblox/RPG Maker/游戏资源 | 资源包、脚本、存档、native 插件和 UI 逻辑哪个是真校验入口 | [mobile-firmware-kernel-and-game-re.md](mobile-firmware-kernel-and-game-re.md)、[go-rust-jvm-and-cpp-reversing.md](go-rust-jvm-and-cpp-reversing.md) |
| Electron/Tauri/Node 桌面应用 | `asar`、JS bundle、native binary、npm package runtime introspection 的边界 | [go-rust-jvm-and-cpp-reversing.md](go-rust-jvm-and-cpp-reversing.md)、[node-and-prototype.md](node-and-prototype.md) |
| Verilog/硬件状态机、SGX enclave、AS/400 SAVF、Glulx | 是否先需要恢复平台格式、状态机或执行模型，再谈 flag 算法 | [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md)、[font-shader-firmware-and-legacy-patterns.md](font-shader-firmware-and-legacy-patterns.md) |

## 合并与拆分结论

- 保留为 family：raw 中平台跨度大，共同价值是先判断“平台/运行时边界”，不是单一算法技巧。
- 不合并进 `mobile-firmware-kernel-and-game-re.md`：本页保留 Android、游戏资源、Electron/Node 和平台运行时的高频 CTF 入口；后者更偏 OS/固件/驱动/底层环境。
- 不拆出 Android 单页：当前 Android 案例还需要同时指向 Frida、anti-analysis、JNI/native、Flutter/Unity 等不同下一跳。

## 常见误判

- 只看 Java/Activity 或 Flutter UI，忽略 native init、JNI 注册、libart hook 和运行时改写。
- 游戏题只搜资源字符串，没有确认资源、脚本、存档和 native 插件之间的数据流。
- Electron/Tauri 只读 JS bundle，没有检查 `child_process`、native addon、asar 解包和平台二进制。
- 硬件/SGX/legacy 平台题先套普通 ELF 逆向，漏掉载体格式和执行环境。

## 关联页面

- [reverse-first-pass-workflow-and-debugging.md](reverse-first-pass-workflow-and-debugging.md)
- [mobile-firmware-kernel-and-game-re.md](mobile-firmware-kernel-and-game-re.md)
- [go-rust-jvm-and-cpp-reversing.md](go-rust-jvm-and-cpp-reversing.md)
- [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md)
- [anti-analysis.md](anti-analysis.md)
- [frida-angr-lldb-and-x64dbg.md](frida-angr-lldb-and-x64dbg.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [WMCTF2025-appfriend-wp](../raw/reverse/WMCTF2025-appfriend-wp.md) | APK 的 Java 层只定位入口，SM4 key/cipher 和 init 段检测在 native so；WP 必须把 key、密文和算法写成可复算数据。 |
| [WMCTF2025-want2become-magicalgirl-wp](../raw/reverse/WMCTF2025-want2become-magicalgirl-wp.md) | Flutter 输入、Java 层魔改 XXTEA、native 层 libart self-hook 和 Flutter 层魔改 AES 串联，先用 smali trace 确认真实执行顺序。 |

## 原始资料

- [android-games-hardware-and-runtime-platforms.md](../raw/reverse/android-games-hardware-and-runtime-platforms.md)
- [WMCTF2025-appfriend-wp](../raw/reverse/WMCTF2025-appfriend-wp.md)
- [WMCTF2025-want2become-magicalgirl-wp](../raw/reverse/WMCTF2025-want2become-magicalgirl-wp.md)
