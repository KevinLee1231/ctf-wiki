---
type: technique
tags: [malware, reverse, powershell, mshta, staged-payload, phishing, obfuscation]
skills: [ctf-malware, ctf-reverse]
raw:
  - ../raw/reverse/Spirit2026-5-im-a-human-wp.md
updated: 2026-05-21
---

# PowerShell Staged Payload and Clipboard Phishing

## 适用场景

题目表面是网页、媒体文件或社会工程提示，实际通过剪贴板命令、`mshta`、HTA/JavaScript、PowerShell 分阶段加载最终载荷。目标不是利用浏览器漏洞，而是把多层混淆脚本还原出来，找到 flag、配置或下一阶段 URL。

## 识别信号

- 页面诱导用户复制命令，命令中出现 `mshta`、`powershell -w hidden`、`-ep Unrestricted`、`iex` 或远程 URL。
- `.mp3`、`.jpg` 等非脚本扩展中出现 `<script>`、`String.fromCharCode`、数字数组、十六进制串或 PowerShell 片段。
- JavaScript 用 `eval`、字符码偏移、XOR 十六进制还原下一阶段；PowerShell 再用变量别名、`Get-Member`、`ExecutionContext.InvokeCommand` 混淆真实调用。
- 最终 payload 可能弹窗、关机、加载 WinForms，flag 夹在 UI 文案或被覆盖前的中间变量里。

## 最小证据

- 已截获首阶段命令或伪装文件，并能证明它会进入 `mshta`、HTA/JS 或 PowerShell 链。
- 至少恢复出一个中间阶段：URL、`String.fromCharCode` 结果、hex/XOR 解码结果或 `ScriptBlock::Create` 内容。
- 能说明哪些动作是纯静态变换，哪些动作有副作用，后者不得直接执行。

## 解法骨架

1. 原样保存入口命令和远程资源；下载只用文本/字节方式，不执行 `mshta` 或 `powershell`。
2. 对 JS 层优先替换 `eval` 为 `console.log` / 写文件，恢复下一阶段文本。
3. 对 PowerShell 层解析 `[char]`、`-bxor`、变量拼接、通配符反射和 `DownloadString`。
4. 每层输出单独保存为 stage 文件，并在文件头记录来源 URL/命令。
5. 搜索最终阶段的 UI 文案、变量覆盖前值、flag-like 字符串和条件分支。

## 关键变体

- **媒体文件藏 HTA/JS。** 扩展名不可信；`mshta <url>` 会按内容解释。
- **PowerShell 反射式混淆。** 通过 `$ExecutionContext.InvokeCommand` 和通配符拼出关键 API。
- **数字表拼接。** `$g='[char]'` 加 `$u=0 ... $l=9`，再把 `$g...+$g...` 拼成源码。
- **诱饵 UI。** 真实 flag 往往在被覆盖前的变量赋值中。

## 常见陷阱

- 在宿主机直接运行入口命令，触发下载、关机、弹窗或污染环境。
- 只保存最终 payload，不保留中间 stage，导致无法解释解题过程。
- 看到 Web 页面就按 `ctf-web` 处理；主障碍是载荷还原时应优先 malware/reverse。
- 忽略伪装扩展名，`.mp3` / `.jpg` 也可能是 HTA/JS 文本。

## 关联技巧

- [scripts-and-obfuscation.md](scripts-and-obfuscation.md)
- [malware-c2-session-key-and-protocol-recovery.md](malware-c2-session-key-and-protocol-recovery.md)
- [self-decrypting-strings-and-lattice-patterns.md](self-decrypting-strings-and-lattice-patterns.md)
- [runtime-patching-oracles-and-tracing.md](runtime-patching-oracles-and-tracing.md)

## 原始资料

- [Spirit2026-5-im-a-human-wp.md](../raw/reverse/Spirit2026-5-im-a-human-wp.md)
