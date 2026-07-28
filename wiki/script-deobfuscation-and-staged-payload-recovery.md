---
type: technique
tags: [malware, reverse, script, deobfuscation, staged-payload]
skills: [ctf-malware, ctf-reverse]
raw:
  - ../raw/malware/scripts-and-obfuscation.md
  - ../raw/malware/UMDCTF2025-suspicious-button-wp.md
updated: 2026-07-28
---

# Script Deobfuscation and Staged-Payload Recovery

## 适用场景

PowerShell/JavaScript/VBA/SVG/包脚本通过字符串拼接、编码、动态求值和多阶段下载隐藏最终配置或载荷；目标是逐层求值并保留每一阶段证据。

## 识别信号

- 大量 Base64/hex、字符数组、压缩流、`eval/Invoke-Expression` 或动态属性。
- 脚本先构造 URL/key，再下载或解密下一阶段。
- 依赖环境、时间、域名或反分析条件选择真实分支。

## 最小证据

- 保存原脚本 hash、每层解码/解密输出和触发条件。
- 禁止真实联网/执行副作用时仍能静态或隔离提取下一阶段。
- 最终配置、URL、key 或载荷可由原逻辑复算。

## 解法骨架

1. 格式化并常量折叠，替换动态 eval 为“打印待执行字符串”。
2. 按数据流逐层解码、解压和解密，给中间产物编号。
3. Stub 网络/文件/进程 API，记录参数而不执行危险动作。
4. 对最终载荷转入对应 PE、脚本或协议分析 technique。

## 关键变体

- PowerShell encoded command 与 AMSI/环境绕过。
- JavaScript/Node 包 install script。
- Office/VBA/HTML/SVG 多层脚本载荷。

## 常见陷阱

- 直接运行未知脚本或允许真实 C2 请求。
- 一次性全自动解混淆，丢失层间 key/条件。
- 只得到下载 URL，未提取后续载荷和配置。

## 关联技巧

- [scripts-and-obfuscation.md](scripts-and-obfuscation.md)
- [staged-loader-and-runtime-image-recovery.md](staged-loader-and-runtime-image-recovery.md)
- [powershell-staged-payload-and-clipboard-phishing.md](powershell-staged-payload-and-clipboard-phishing.md)

## 原始资料

- [scripts-and-obfuscation.md](../raw/malware/scripts-and-obfuscation.md)
- [UMDCTF2025-suspicious-button-wp](../raw/malware/UMDCTF2025-suspicious-button-wp.md)
