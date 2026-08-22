---
type: tooling
tags: [mobile, tooling, tools, environment, android, ios]
skills: [ctf-mobile]
---

# Mobile Tooling

本页是 `ctf-mobile` 方向本机工具信息的唯一权威来源，维护当前安装状态、版本、路径、环境、完整调用、适用边界和失败处理。`ctf-mobile/SKILL.md` 只说明何时选择某类工具或知识页，不复制本页细节。

本页只描述当前真实状态；实际环境与本文不一致时，直接修正文中现状，不在本页累积核验记录或旧版本历史。

## 工具选择边界

- APK/IPA 先做离线 Manifest/plist/entitlement/DEX/native 清单；只有组件、IPC、权限、签名、Keystore/Keychain 或平台运行时决定解法时才留在 Mobile。
- 动态 hook 只使用题目模拟器、一次性测试设备或明确授权环境，不默认操作个人设备和真实应用数据。
- 普通算法/字节码/native 语义转 Reverse；已确认 native 内存破坏原语转 Pwn。

## 完整调用约定

WSL 系统工具用绝对路径；Frida 使用 `ctf-tools`；Windows JADX 由 `pwsh` 直接运行：

```pwsh
wsl /usr/bin/apktool d /path/to/app.apk -o /path/to/decoded
wsl /usr/bin/baksmali d /path/to/classes.dex -o /path/to/smali
& "D:/CTF工具/jadx-1.5.6/bin/jadx.bat" -d "D:/path/to/jadx-output" "D:/path/to/app.apk"
wsl /home/kali/miniforge3/bin/conda run --no-capture-output -n ctf-tools frida -U -f com.example.app -l /path/to/hook.js
```

## 当前状态与路径

| 工具 | 当前状态/版本 | 路径或环境 | 何时使用 | 完整调用 |
|---|---|---|---|---|
| `apktool` | 可用，2.7.0-dirty | `/usr/bin/apktool`，WSL system | Android resources、Manifest 与 smali 解包 | 见上方命令 |
| `baksmali` | 可用，2.5.2 | `/usr/bin/baksmali`，WSL system | DEX 字节码和关键分支的低层核验 | 见上方命令 |
| JADX | 可用，1.5.6 | `D:/CTF工具/jadx-1.5.6/bin/jadx.bat`；GUI 为同目录 `jadx-gui.bat` | Java/Kotlin-like 反编译、搜索和 xref | CLI 见上方；GUI：`& "D:/CTF工具/jadx-1.5.6/bin/jadx-gui.bat" "D:/path/to/app.apk"` |
| Manifest summary | 可用 | `C:/Users/LMY/.agents/skills/ctf-reverse/scripts/manifest-summary.ps1` | 已解码 Manifest 的 SDK、权限、组件和入口摘要 | `& "C:/Users/LMY/.agents/skills/ctf-reverse/scripts/manifest-summary.ps1" -ManifestPath "D:/path/to/decoded/AndroidManifest.xml"` |
| Frida / frida-tools | 可用，17.9.1 / 14.8.1 | WSL `ctf-tools` | Java/ObjC/native hook、参数观察和临时 patch | 见上方命令；进程首检用 `wsl /home/kali/miniforge3/bin/conda run --no-capture-output -n ctf-tools frida-ps -U` |
| IDA Pro MCP（`idalib`） | 方法可调用；当前 0 个会话；版本未暴露 | MCP | APK/IPA 内 native library、Mach-O/ELF 与 JNI 边界 | `idb_list` → `idb_open(绝对路径)` → `survey_binary` |
| Ghidra MCP | 方法可调用；当前无运行实例；版本未暴露 | MCP | IDA 不适用或已有 Ghidra 项目时的 native 分析 | `list_instances` → `connect_instance` |
| `adb` | WSL 与 Windows `PATH` 均未发现 | 未安装/未配置 | 组件调用、日志、文件传输和设备状态 | 当前无可执行命令 |
| `lldb` | WSL 与 Windows `PATH` 均未发现 | 未安装/未配置 | iOS/Mach-O/native 运行时调试 | 当前无可执行命令 |

## 失败处理

- JADX 伪代码失真或函数失败：回到 `baksmali`/DEX 或 native 汇编核验，不把高层视图当作唯一证据。
- 没有 `adb`：先完成静态组件、权限、deep link 和 WebView 数据流分析；确认动态设备链确实必要后再决定安装 platform-tools。
- Frida 连接失败：先确认设备、ABI、API level、server 版本、权限和进程启动时机；当前只有本机 CLI，不代表设备侧 server 已配置。
- `lldb` 缺失且题目必须做 iOS 运行时调试：先保留 Mach-O、entitlement 和静态证据，再为题目建立独立 Apple 调试环境；不要把 Android 工具硬套到 iOS。

## 关联知识页

- [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)
- [mobile-firmware-kernel-and-game-re.md](mobile-firmware-kernel-and-game-re.md)
- [mobile-webview-url-scheme-native-bridge.md](mobile-webview-url-scheme-native-bridge.md)
- [android-debuggable-run-as-private-data.md](android-debuggable-run-as-private-data.md)
- [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md)
