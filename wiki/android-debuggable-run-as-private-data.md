---
type: technique
tags: [mobile, android, debuggable, adb, run-as, private-data, technique]
skills: [ctf-mobile]
raw:
  - ../raw/mobile/SekaiCTF2026-SekaiID-wp.md
updated: 2026-07-27
---

# Android Debuggable, run-as and Private Data

## 适用场景

Android 应用把 secret 放在 `0600` 私有文件、SharedPreferences 或数据库中，并依赖应用 UID 隔离。但发布包仍带 `android:debuggable="true"`，攻击者可通过 ADB `run-as <package>` 切换到该应用 UID，直接读取应用私有目录。全局 `adb root` 被禁用并不能消除这个应用级调试能力。

## 识别信号

- Manifest 或 `aapt dump badging` 显示 debuggable。
- 设备允许 ADB shell，`run-as <package> id` 成功。
- 目标数据归应用 UID 所有，普通 shell 读取失败。
- 题目强调文件权限、Provider/Intent 或签名校验，但最终读取只需要获得应用 UID。

## 最小证据

- 确认目标包名、UID、debuggable 标志和应用数据目录。
- 对比普通 `shell` 与 `run-as` 后的 `id`、目录权限和文件读取结果。
- 证明读取依赖应用调试身份，而不是设备 root、内核漏洞或普通路径穿越。
- 若还涉及 Intent/Provider，单独记录它们是否为前置落点还是无关旁路。

## 解法骨架

1. 从 APK/设备提取 manifest 和包名，核对实际安装版本的 debuggable 状态。
2. 执行 `adb shell run-as <package> id` 验证 UID 切换。
3. 在该身份下枚举 `files/`、`shared_prefs/`、`databases/`、`cache/` 等私有目录。
4. 精确读取目标文件或数据库，并保留普通 shell 失败的对照。
5. 若数据还受应用层加密保护，再回到 Keystore/密钥派生和业务代码分析。

## 关键变体

- `run-as` 依赖包可调试和系统支持；失败时先区分 release build、包名错误和设备策略。
- 文件模式 `0600` 只限制其它 UID，不限制通过 `run-as` 获得的同 UID 身份。
- `adb root` 控制 adbd 是否为 root，与应用是否可调试是不同边界。
- exported component、隐式 Intent 或 Provider grant 可能提供另一条路径，但不应掩盖调试标志这一决定性原语。

## 常见陷阱

- 看到 `Permission denied` 就转向内核提权，没有检查 debuggable。
- 把 `adb root` 失败当作 `run-as` 也不可用。
- 只审计源码 manifest，没有核对最终安装 APK 的合并 manifest。
- 读到缓存或测试数据就停止，没有确认目标文件与当前用户/包 UID 一致。

## 关联技巧

- [mobile-firmware-kernel-and-game-re.md](mobile-firmware-kernel-and-game-re.md)
- [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)
- [mobile-webview-url-scheme-native-bridge.md](mobile-webview-url-scheme-native-bridge.md)
- [reverse-tooling.md](reverse-tooling.md)

## 原始资料

- [SekaiCTF2026-SekaiID-wp.md](../raw/mobile/SekaiCTF2026-SekaiID-wp.md)
