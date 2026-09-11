---
type: tooling
tags: [mobile, tooling, tools, environment, android, ios]
skills: [ctf-mobile]
---

# Mobile Tooling

本页服务 Android/iOS 组件、IPC、权限、签名、WebView、Keystore/Keychain 和平台沙箱问题。普通 APK 算法或 native 逻辑还原仍由 Reverse 主导。

## 平台与工具状态

APK/DEX 静态阅读和一般 Python 分析优先 Windows。JADX、Java、Manifest summary、IDA/Ghidra 的当前路径与完整调用统一见 [reverse-tooling.md](reverse-tooling.md)。文件存在或反编译成功不能证明设备、模拟器、USB、调试权限或签名条件已就绪。

| 工具 | 当前状态与位置 | 使用条件 |
|---|---|---|
| JADX / Manifest summary | Windows 已有；使用 Reverse 工具页入口 | 先读组件、权限、Intent 和 Java/Kotlin-like 逻辑 |
| Windows Python | `D:/文档/新建文件夹/venv/Scripts/python.exe`，3.14.3 | 解析已导出的 XML/JSON、资源、密文与脚本数据 |
| apktool | WSL `/usr/bin/apktool`；APT `2.7.0+dfsg-7.2` | 已有 Linux 工具，用于低层 APK 解包、资源和 smali；一般静态阅读先用 Windows JADX |
| baksmali | WSL `/usr/bin/baksmali`；APT `libsmali-java 2.5.2.git2771eae-4` | DEX 指令、寄存器与控制流核验 |
| Frida | WSL `ctf-tools` 原生插桩入口保留，详见 Reverse 工具页 | 已验证 Linux 本地 CLI；不能据此认定 Android/iOS 连接可用 |

当前 Windows 与 WSL PATH 均未发现 `adb`，原 APKiD、Androguard 独立入口已不存在。本页没有已验证的移动设备/server 接入链；不能直接执行旧的 `apkid`、`androguard` 或设备命令。移动端所需 Python 包或 SDK 优先评估 Windows 兼容版本，只有实际设备/运行能力要求 Linux 时才考虑 WSL。

## 从 pwsh 调用

先通过 Reverse 工具页的 Windows JADX 命令读取 APK；需要现有低层解包器时：

```pwsh
wsl -d kali-linux --exec /usr/bin/apktool d "/mnt/d/题目路径/app.apk" -o "/mnt/d/题目路径/apk-decoded"
wsl -d kali-linux --exec /usr/bin/baksmali d "/mnt/d/题目路径/classes.dex" -o "/mnt/d/题目路径/smali-output"
& "D:/文档/新建文件夹/venv/Scripts/python.exe" "D:/题目路径/analyze_mobile.py"
```

输出目录属于本题工作区；`/mnt/d/题目路径` 对应 Windows `D:/题目路径`。`apktool` 与 `baksmali` 是 APT 入口，不放进 Conda。解包后用实际生成的文本 XML 交给 Manifest summary，不能直接把二进制 AXML 当 UTF-8。

动态分析先确定本题目标设备、ABI、Android/iOS 版本、授权状态与可用连接，再核对 client/server 版本、进程和 attach/spawn 条件。需要部署 server、SDK 或其他软件时另行说明安装范围；已有 Frida 客户端不构成部署授权。

## 失败处理与下一跳

- JADX 伪代码失败或 exported/权限结论冲突：对照 Manifest、DEX/smali 和实际平台版本；见 [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)。
- apktool 旧资源格式支持不足：保留原 APK，先检查 Windows JADX 输出；升级工具前按安装规则说明，不自动修复 WSL 依赖。
- WebView、URL Scheme、JS bridge 决定权限边界：转 [mobile-webview-url-scheme-native-bridge.md](mobile-webview-url-scheme-native-bridge.md)。
- `run-as`、私有目录或 debuggable 条件不满足：核对 manifest、签名与设备实际状态，见 [android-debuggable-run-as-private-data.md](android-debuggable-run-as-private-data.md)。
- Hook 不命中：先核对进程、ABI、加载时机和连接；运行时观测看 [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md)，平台分流看 [mobile-firmware-kernel-and-game-re.md](mobile-firmware-kernel-and-game-re.md)。
- WSL 新增、升级、重装须先说明用途、为何必须 Linux、目标、方式及依赖影响并取得明确同意。
