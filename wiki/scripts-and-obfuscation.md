---
type: family
tags: [malware, family, scripts, obfuscation, staged-payload, c2]
skills: [ctf-malware]
raw:
  - ../raw/malware/scripts-and-obfuscation.md
  - ../raw/pentest/WMCTF2025-shopping-company-phishing-email-wp.md
updated: 2026-06-12
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

## 合并与拆分结论

本页应为 family。脚本混淆、PowerShell stage、包投毒、shellcode loader、反沙箱和 IOC 提炼共享“载荷链分阶段还原”的首轮模型，但每个具体技术已由相邻页面承接。当前不删除，是因为 raw 和 WP 同时覆盖了恶意脚本、包载荷和 SVG 邮件附件；当前不拆分，是因为拆分会让短案例失去统一入口。

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [WMCTF2025-shopping-company-phishing-email-wp](../raw/pentest/WMCTF2025-shopping-company-phishing-email-wp.md) | `.eml` 发票催收附件中的 SVG 同时承载视觉诱饵和 JavaScript；分析时先解 MIME/base64，再还原脚本映射表、XOR/key 和诱饵 flag。 |
| [D3CTF2021-baby-spear-wp](../raw/reverse/D3CTF2021-baby-spear-wp.md) | 隐藏 VBA 宏释放 PE 并用时间派生 AES key，先恢复 Office 宏流和 staged payload。 |

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
