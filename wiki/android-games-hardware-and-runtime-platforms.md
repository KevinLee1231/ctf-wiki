---
type: family
tags: [reverse, family, android, games, runtime-platforms]
skills: [ctf-mobile, ctf-reverse, ctf-hardware-embedded]
raw:
  - ../raw/reverse/android-games-hardware-and-runtime-platforms.md
  - ../raw/reverse/WMCTF2025-appfriend-wp.md
  - ../raw/reverse/WMCTF2025-want2become-magicalgirl-wp.md
updated: 2026-07-27
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

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| 游戏资源、scene、save 或客户端/服务端状态含目标 | [game-asset-and-scene-state-extraction.md](game-asset-and-scene-state-extraction.md) |
| WebView/URL Scheme/native bridge 是移动平台攻击面 | [mobile-webview-url-scheme-native-bridge.md](mobile-webview-url-scheme-native-bridge.md) |
| debuggable/run-as/backup 暴露 Android 私有数据 | [android-debuggable-run-as-private-data.md](android-debuggable-run-as-private-data.md) |

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
- [mobile-webview-url-scheme-native-bridge.md](mobile-webview-url-scheme-native-bridge.md)
- [android-debuggable-run-as-private-data.md](android-debuggable-run-as-private-data.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [WMCTF2025-appfriend-wp](../raw/reverse/WMCTF2025-appfriend-wp.md) | APK 的 Java 层只定位入口，SM4 key/cipher 和 init 段检测在 native so；WP 必须把 key、密文和算法写成可复算数据。 |
| [WMCTF2025-want2become-magicalgirl-wp](../raw/reverse/WMCTF2025-want2become-magicalgirl-wp.md) | Flutter 输入、Java 层魔改 XXTEA、native 层 libart self-hook 和 Flutter 层魔改 AES 串联，先用 smali trace 确认真实执行顺序。 |
| [D3CTF2019-machine-wp](../raw/reverse/D3CTF2019-machine-wp.md) | Android/Frida 运行时平台是主边界，先确认 Java/native 调用和动态 hook 位置。 |
| [D3CTF2021-no-name-wp](../raw/reverse/D3CTF2021-no-name-wp.md) | Android assets 加密 Java 校验代码由 native 返回 key，先恢复运行时真实验证接口。 |
| [D3CTF2022-d3hotel-wp](../raw/reverse/D3CTF2022-d3hotel-wp.md) | Unity WebGL/IL2CPP/WASM 和 Lua 假 flag 组合，先建立 metadata、Lua、wasm 函数映射。 |
| [D3CTF2022-d3mug-wp](../raw/reverse/D3CTF2022-d3mug-wp.md) | Unity3D Android 游戏可通过 Frida 调用 native update/get，先拆资源谱面并复用外部函数。 |
| [D3CTF2023-d3tetris-wp](../raw/reverse/D3CTF2023-d3tetris-wp.md) | Android/Java 游戏和 native 校验混合，先拆资源/运行时调用再恢复游戏状态或算法。 |
| [D3CTF2025-d3piano-wp](../raw/reverse/D3CTF2025-d3piano-wp.md) | Android 钢琴 app 的 native hook 链和 Salsa20/LZW 音符序列，先识别 Frida/gumpp 篡改点。 |
| [D3CTF2025-d3rpg-revenge-wp](../raw/reverse/D3CTF2025-d3rpg-revenge-wp.md) | RPG Maker/RGSS 资源包魔改和 Ruby 脚本明文内存，先脱壳或搜索脚本恢复校验。 |
| [HGAME2026-androuge-wp](../raw/reverse/HGAME2026-androuge-wp.md) | APK 释放魔改 Lua 5.4 VM 与加密 `game` bytecode；先还原 XOR 载入层和 opcode 位域，再提密文数组与 seed。 |
| [HGAME2026-vidarchall-wp](../raw/reverse/HGAME2026-vidarchall-wp.md) | Android zygote preload、isolatedProcess 和 native 埋点共同影响 XXTEA 参数，先确认多进程运行时状态。 |
| [LilacCTF2026-nineapple-wp](../raw/reverse/LilacCTF2026-nineapple-wp.md) | iOS Swift 九宫格手势锁给出 `weight/target_all/map_list`；无需操作 UI，按加权和反查每个字符路径。 |
| [NCTF2026-nomybank-wp](../raw/reverse/NCTF2026-nomybank-wp.md) | Godot PCK 密钥、运行时解密 DLL、TLS callback hook 和 SMC 共同隐藏真实校验；先恢复资源和动态补丁链。 |
| [NCTF2026-pay-for-2048-wp](../raw/reverse/NCTF2026-pay-for-2048-wp.md) | Electron `app.asar` 中 JS bridge 调 WASM license 校验；先直接用 Node 调应用服务，再补 WASM digest/key 派生。 |
| [SUCTF2026-easygalWP](../raw/reverse/SUCTF2026-easygalWP.md) | Unity/IL2CPP 资源中反序列化 Story 节点；恢复 choice 的 weight/value/marker 后建模为带路径恢复的背包 DP。 |
| [SUCTF2026-flumelWP](../raw/reverse/SUCTF2026-flumelWP.md) | Flutter/Dart 输入先经 `Rc4Warp`，再由新版 `libjunk.so` 验证 Hermes bundle 并派生 AES-CBC key/IV；旧 placeholder 会误导。 |
| [SUCTF2026-MvsicPlayerWP](../raw/reverse/SUCTF2026-MvsicPlayerWP.md) | Electron 音乐播放器先解析 `.su_mv` payload，再由 native `.node` 对 WAV 分支执行 VM bytecode 加密；目标是恢复原 WAV MD5。 |
| [VNCTF2026-login-wp](../raw/reverse/VNCTF2026-login-wp.md) | APK Java 层只组包，native so 完成魔改 AES、HTTP header 签名和 Frida/IDA 环境检测；可结合流量复算。 |
| [MoeCTF2024-SecretModule-wp](../raw/mobile/MoeCTF2024-SecretModule-wp.md) | 这题的有效分析顺序是先识别 Magisk 模块入口，再解开 Shell 包装，最后把物理按键交互抽象成二元枚举。`getevent -lc 1` 每次只读取一个输入事件；七次选择空间极小，MD5 在这里仅是验证函数，并不需要进行通用哈希破解。 |
| [UMDCTF2020-droided-chungus-wp](../raw/mobile/UMDCTF2020-droided-chungus-wp.md) | 本题按现存仓库只能走服务端源码复原路线，不能假装完成了已经缺失的 APK 迷宫逆向。服务协议的关键是准确复刻带 32 位溢出的移位异或和打乱过程，再按服务端保存的方向序列推进，最后解 Base85。 |
| [UMDCTF2023-pokeball-escape-wp](../raw/mobile/UMDCTF2023-pokeball-escape-wp.md) | 同时跟踪 Android 系统属性检查、JNI 方法和资源解密逻辑；仅关闭应用并不等于题目要求的 escape；动态路线：Hook `systemInfo()` 或修补分支，让应用相信设备厂商是 `Devon Corporation`。 |
| [0xGame2023-week3-虚构核心-wp](../raw/reverse/0xGame2023-week3-虚构核心-wp.md) | 该题的关键是沿动态加载链找到真正的校验代码：外层 APK 只负责用固定密钥循环异或解密 DEX，内层 `FlagChecker` 才包含 flag 结构和 MD5 条件。四位十六进制字符串的搜索空间很小，直接本地穷举既可复现，也避免依赖外部反查结果。 |
| [ACTF2023-native-app-wp](../raw/reverse/ACTF2023-native-app-wp.md) | 本题的逆向难点有两层：先从 Flutter AOT 而不是 DEX 中恢复 Dart 业务逻辑，再正确理解多个 UI 回调之间的状态依赖。算法本身是标准 RC4，加密前多了一次逐字节异或 `0xff`，利用 RC4 对称性即可直接反解。 |
| [SekaiCTF2026-ufo-wp](../raw/reverse/SekaiCTF2026-ufo-wp.md) | 这是一道 APK 状态逆向与 QR 重建题，决定性障碍不是 Android 权限或付费逻辑，因此归入 Reverse 比仓库原先的 Misc 更准确。面对名称混淆，应优先追踪仍可读的偏好键、方法签名和状态数据流；识别出 $21\times21$ 网格后，也不能因为缺少三个定位框就否定 QR，而应按版本补齐功能模块，再由格式信息或枚举确定纠错等级与 mask。 |
| [UMDCTF2023-jnidorino-wp](../raw/reverse/UMDCTF2023-jnidorino-wp.md) | 将托管层 native 声明、实际调用和 ELF 导出符号视为三个集合，优先做集合差分；大量同构、随机命名的 JNI 包装通常是噪声；题目若提示“漏掉一个函数”，应先比较数量与名称，而不是逐函数逆向。 |
| [UMDCTF2023-playtime-wp](../raw/reverse/UMDCTF2023-playtime-wp.md) | 从地图头指针表和异常地图常量定位隐藏内容，而不是盲目通关整部游戏；“地图存在但没有 warp”是典型不可达资源信号，可通过调试器改地图 ID 或补丁已有入口访问。 |
| [WMCTF2023-ezandroid-wp](../raw/reverse/WMCTF2023-ezandroid-wp.md) | 对 goron 控制流平坦化和字符串加密的 Android native 库，先 hook `RegisterNatives` 定位 JNI 函数，再绕过 Frida 检测并动态抓 RC4/AES 的输入输出。 |
| [WMCTF2024-re1-wp](../raw/reverse/WMCTF2024-re1-wp.md) | 识别点：导出表函数和实际 JNI 注册函数不一致；分析方式：从 `JNI_OnLoad/RegisterNatives` 反查真实 native 方法。 |

## 原始资料

- [android-games-hardware-and-runtime-platforms.md](../raw/reverse/android-games-hardware-and-runtime-platforms.md)
- [WMCTF2025-appfriend-wp](../raw/reverse/WMCTF2025-appfriend-wp.md)
- [WMCTF2025-want2become-magicalgirl-wp](../raw/reverse/WMCTF2025-want2become-magicalgirl-wp.md)
