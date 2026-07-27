---
type: family
tags: [malware, family, scripts, obfuscation, staged-payload, c2]
skills: [ctf-malware]
raw:
  - ../raw/malware/scripts-and-obfuscation.md
  - ../raw/pentest/WMCTF2025-shopping-company-phishing-email-wp.md
updated: 2026-07-27
---

# Scripts and Obfuscation Analysis

## 作用边界

本页是 Malware 脚本载荷与混淆 family 页，负责从 JS、PowerShell、Office 宏、SVG/HTML 附件、包管理器脚本、shellcode loader、API hashing、反沙箱和动态行为中分流到具体恢复路线。

它不是脚本解混淆 technique 的大杂烩。首轮要判断载荷链的阶段：文本混淆、编码壳、下载器、配置提取、C2 协议、进程注入、内存驻留还是 IOC/YARA 提炼。不同阶段需要的证据和工具完全不同。

## 识别信号

- 样本包含 `.js`、`.ps1`、宏、`.eml`、SVG、HTML、npm/deb/plugin、shellcode blob、Base64/hex/charcode、`Invoke-Expression`、`eval`、WMI、注册表或计划任务。
- 代码中存在分阶段下载、字符串表、异或/自定义 alphabet、环境探测、反沙箱、API hashing、C2 URL、User-Agent、session key 或加密配置。
- 运行样本会改文件、进程、注册表、剪贴板、网络连接或持久化项，但直接执行风险过高。
- 目标通常是恢复 payload、配置、IOC、通信密钥或藏在诱饵中的 flag。

## 最小证据

- 原始载荷入口和编码层级：MIME/base64、压缩包、XML/SVG CDATA、PowerShell encoded command、JS string table 或包安装脚本。
- 每一阶段的输入输出：解码字符串、下载 URL、key、payload hash、落盘路径、进程名、网络域名。
- 能在隔离环境或静态模拟中验证解混淆结果，而不是直接运行未知样本。
- 能判断下一步是脚本恢复、C2 协议、PE/.NET、内存取证、YARA/IOC，还是普通 Crypto 表示层编码。

## 首轮路由

| 证据形态 | 首轮判断 | 下一跳 |
|---|---|---|
| SVG/HTML/邮件附件含脚本、CDATA、自动跳转或诱饵 flag | 先解 MIME 和 XML/脚本层，区分视觉诱饵、真实脚本和二阶段载荷 | [exotic-encodings-and-file-formats.md](exotic-encodings-and-file-formats.md) |
| JavaScript string table、`eval`、charcode、控制流平坦化 | 先静态还原字符串和调用图，必要时用受控解释器替换危险 API | [malware-tooling.md](malware-tooling.md) |
| PowerShell encoded command、clipboard hijack、download cradle | 先解码命令和阶段边界，再提取 URL、key、落盘路径和执行策略 | [powershell-staged-payload-and-clipboard-phishing.md](powershell-staged-payload-and-clipboard-phishing.md) |
| Debian/npm/plugin 或 trojanized package | 先看 metadata、install/postinst、依赖脚本和二进制扩展，再还原 C2 配置 | [malware-c2-session-key-and-protocol-recovery.md](malware-c2-session-key-and-protocol-recovery.md) |
| Shellcode、API hashing、process injection、loader | 先恢复 API 解析和内存映射，再判断是否转 PE/.NET 或 pwn shellcode 分析 | [pe-and-dotnet.md](pe-and-dotnet.md), [windows-arm-and-cross-platform-exploits.md](windows-arm-and-cross-platform-exploits.md) |
| 反沙箱、主机名、环境变量、时间延迟 | 先伪造环境或 patch 检测点，避免在错误分支提取配置 | [anti-analysis.md](anti-analysis.md) |
| 需要输出 IOC、YARA 或行为报告 | 先固定已验证的域名、路径、mutex、key、hash 和协议字段，不把猜测写成 IOC | [malware-tooling.md](malware-tooling.md) |

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| PowerShell/JS/VBA/包脚本多层编码、eval 和下载链 | [script-deobfuscation-and-staged-payload-recovery.md](script-deobfuscation-and-staged-payload-recovery.md) |
| PowerShell 分阶段载荷与剪贴板/钓鱼行为需要复原 | [powershell-staged-payload-and-clipboard-phishing.md](powershell-staged-payload-and-clipboard-phishing.md) |
| 脚本最终恢复 C2 配置、session key 或恶意协议 | [malware-c2-session-key-and-protocol-recovery.md](malware-c2-session-key-and-protocol-recovery.md) |

## 合并与拆分结论

本页应为 family。脚本混淆、PowerShell stage、包投毒、shellcode loader、反沙箱和 IOC 提炼共享“载荷链分阶段还原”的首轮模型；通用脚本去混淆、PowerShell 钓鱼链和 C2 协议恢复分别由相邻 technique 承接，低频载体继续保留统一入口。

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [WMCTF2025-shopping-company-phishing-email-wp](../raw/pentest/WMCTF2025-shopping-company-phishing-email-wp.md) | `.eml` 发票催收附件中的 SVG 同时承载视觉诱饵和 JavaScript；分析时先解 MIME/base64，再还原脚本映射表、XOR/key 和诱饵 flag。 |
| [D3CTF2021-baby-spear-wp](../raw/reverse/D3CTF2021-baby-spear-wp.md) | 隐藏 VBA 宏释放 PE 并用时间派生 AES key，先恢复 Office 宏流和 staged payload。 |
| [0xGame2024-week4-Encrypted-file-wp](../raw/malware/0xGame2024-week4-Encrypted-file-wp.md) | 本题的证据链是“上传 WebShell → 从 WebShell 恢复固定通信算法 → 解密命令流 → 找到 OpenSSL 加密命令 → 用同一参数逆向解密附件”。分析冰蝎流量时，不必逐段粘贴超长密文；保留协议解码逻辑、完整攻击命令序列以及最终文件恢复命令，WP 即可独立复现。 |
| [UMDCTF2018-excellent-security-wp](../raw/malware/UMDCTF2018-excellent-security-wp.md) | 分析脚本恶意代码时要同时记录执行、落地和持久化三条行为链，并依据实际解释器判断转义语义。本题若机械删除所有 `^`，会得到错误 flag；字符是否具有转义作用取决于它出现在哪一层语言和哪种字符串上下文中。 |
| [UMDCTF2019-oh-squib-wp](../raw/malware/UMDCTF2019-oh-squib-wp.md) | 分析 Office 恶意文档时，应优先离线解包并读取 XML、关系文件和字段代码，避免直接打开并启用内容。视觉隐藏不会从 OOXML 中删除命令。对外链载荷要把关键命令链和解码数据完整归纳进正文；临时 Pastebin 一类地址既非复现所必需，也可能失效或带来执行风险，因此不保留。 |
| [UMDCTF2025-suspicious-button-wp](../raw/malware/UMDCTF2025-suspicious-button-wp.md) | 静态化简 QML 中的动态属性访问和字符置换，恢复下载命令，再对留存的 Base64/bzip2 第二阶段做离线解码；桌面主题或 plasmoid 引入 `executable` 数据源、点击事件中出现混淆字符串、远程脚本直接管道到 shell。 |
| [WMCTF2024-party-time-wp](../raw/malware/WMCTF2024-party-time-wp.md) | 宏分析重点：文档属性 `Comments` 中藏 payload，不在宏正文直接给命令；恶意程序关键：私钥也存入注册表，真正需要补的是 OAEP label。 |
| [0xGame2025-week3-ezVBS-wp](../raw/reverse/0xGame2025-week3-ezVBS-wp.md) | 本题由“算术构造字符 → `Execute` 动态执行 → 可打印字符轮转 → 覆盖自身 → 自定义 Base64”组成。危险点不在最终编码，而在前几层具有真实副作用：若直接执行原脚本，旧层会被覆盖，证据链随之丢失。 |
| [UMDCTF2017-death-gripes-wp](../raw/reverse/UMDCTF2017-death-gripes-wp.md) | 动态语言混淆经常把简单常量藏进语法噪声。先识别 `.call` 链的稳定状态转移，就能把整个程序简化成一元计数编码。静态计数比直接 `eval` 未知数据区更安全，也更容易说明结果为何成立。 |

## 常见陷阱

- 为了看行为直接执行未知脚本，导致环境被改写或网络请求污染证据。
- 只解第一层 Base64，没继续追踪下载器、二阶段 payload 和配置 blob。
- 把诱饵 flag 当结果，忽略脚本里的真实解密路径。
- 恢复 C2 字符串后不验证 session key、编码表和协议字段。
- 把普通编码题误归 Malware；没有恶意行为、配置或载荷链时应转 Crypto，隐藏载荷则转 Stego。

## 关联技巧

- [powershell-staged-payload-and-clipboard-phishing.md](powershell-staged-payload-and-clipboard-phishing.md)
- [malware-c2-session-key-and-protocol-recovery.md](malware-c2-session-key-and-protocol-recovery.md)
- [pe-and-dotnet.md](pe-and-dotnet.md)
- [anti-analysis.md](anti-analysis.md)
- [exotic-encodings-and-file-formats.md](exotic-encodings-and-file-formats.md)
- [malware-tooling.md](malware-tooling.md)

## 原始资料

- [scripts-and-obfuscation.md](../raw/malware/scripts-and-obfuscation.md)
- [WMCTF2025-shopping-company-phishing-email-wp](../raw/pentest/WMCTF2025-shopping-company-phishing-email-wp.md)
