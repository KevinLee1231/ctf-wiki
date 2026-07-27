---
type: technique
tags: [mobile, webview, url-scheme, javascript-bridge, native, technique]
skills: [ctf-mobile]
raw:
  - ../raw/mobile/WMCTF2022-chess-wp.md
  - ../raw/mobile/WMCTF2022-wm-baby-droid-wp.md
updated: 2026-07-27
---

# Mobile WebView URL Scheme and Native Bridge

## 适用场景

Android/iOS 应用通过自定义 URL Scheme、deep link 或外部 Intent 打开 WebView，并向页面注入原生 JavaScript bridge。若 URL 信任、下载落点或页面来源校验可绕过，攻击者脚本就可能调用敏感原生方法、读取应用私有数据或加载被覆盖的 native library。

## 识别信号

- Manifest/Info.plist 注册 scheme、deep link、exported Activity 或 URL handler。
- WebView 启用 JavaScript，并使用 `addJavascriptInterface`、message handler 或自定义 bridge。
- 信任判断只比较 scheme/前缀，允许 `data:`、重定向、编码或嵌套 URL。
- 下载回调使用响应文件名拼接内部路径，随后 `System.load` 或原生方法访问该路径。

## 最小证据

- 还原从外部 URL/Intent 到最终 WebView URL 的完整改写和 decode 顺序。
- 列出 bridge 暴露的方法、线程、参数和调用时的应用权限。
- 证明攻击者页面在目标 WebView 上下文中执行，而非普通外部浏览器。
- 涉及文件覆盖时确认实际落点、写入完成时机和后续加载点。

## 解法骨架

1. 静态定位 scheme/deep-link 入口、WebView 配置、导航白名单和 bridge 注册。
2. 动态记录每次重写、跳转和页面 origin，找出能进入受信 WebView 的攻击者内容。
3. 构造最小脚本调用一个无害 bridge 方法，验证 JS 到 native 的能力边界。
4. 若需读私有文件或加载库，先建立同 UID 下的写入/读取原语，再处理路径穿越和竞态。
5. 对脚本、URL 和响应头分别按所在解析层编码，并验证远端启动顺序。

## 关键变体

- iOS `data:` URL 可能绕过只按 URL 类型分支的可信判断。
- Android 下载文件名可将外部响应写入应用私有目录，再由 bridge 触发 `System.load`。
- bridge 只暴露有限方法时，重点检查这些方法是否间接访问文件、反射或 native 层。
- 反调试只是分析障碍，需先 patch/绕过再确认真实入口和数据流。

## 常见陷阱

- 看到 WebView 就按普通 XSS 处理，忽略脚本运行在应用 UID 和原生 bridge 权限下。
- 只验证 deep link 能打开页面，没有确认页面被哪个 WebView 和 bridge 接收。
- 多次 URL 编码凭经验叠加，导致过滤器和最终页面看到不同 payload。
- 文件尚未写完就触发 native load，出现不稳定崩溃。

## 关联技巧

- [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)
- [mobile-firmware-kernel-and-game-re.md](mobile-firmware-kernel-and-game-re.md)
- [xss-dom-and-browser-tricks.md](xss-dom-and-browser-tricks.md)
- [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md)

## 原始资料

- [WMCTF2022-chess-wp.md](../raw/mobile/WMCTF2022-chess-wp.md)
- [WMCTF2022-wm-baby-droid-wp.md](../raw/mobile/WMCTF2022-wm-baby-droid-wp.md)
