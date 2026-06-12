---
type: technique
tags: [malware, technique]
skills: [ctf-malware]
raw:
  - ../raw/malware/scripts-and-obfuscation.md
updated: 2026-05-21
---

# Scripts and Obfuscation Analysis

## 适用场景

恶意行为、脚本载荷链、C2、配置提取、反分析或样本行为还原是主要障碍。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 样本包含混淆脚本、下载器、Office 宏、PE/.NET、C2 字符串或 IOC。
- 存在字符串解密、配置 blob、反调试、反沙箱或网络 beacon。
- 目标是提取配置、payload、flag 或通信密钥。
- 题面或 raw 线索能落到这些关键词之一：Obfuscated Scripts (General)、JavaScript Deobfuscation、PowerShell Analysis、Junk Code Detection、Hex-Encoded Payloads、Debian Package Analysis、Dynamic Analysis Techniques、YARA Rules for Malware Detection。

## 最小证据

- 已完成主方向判断，并确认本页技巧比相邻技巧更能解释当前证据。
- 至少有一个可复现输入、输出、文件结构、数学关系、协议行为或运行时状态。
- 能指出 raw 案例中哪一个变体与当前题最接近，以及不同点在哪里。

## 解法骨架

1. 优先静态拆层。
2. 恢复每一阶段脚本、URL、key、payload 和配置。
3. 必要时只在隔离环境动态验证。
4. 输出 IOC、配置和最终 flag 来源。

## 关键变体

| 变体 | 复用重点 |
|---|---|
| Obfuscated Scripts (General) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| JavaScript Deobfuscation | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| PowerShell Analysis | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Junk Code Detection | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Hex-Encoded Payloads | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Debian Package Analysis | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Dynamic Analysis Techniques | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| YARA Rules for Malware Detection | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Shellcode Analysis | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Memory Forensics for Malware | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Anti-Analysis Techniques | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| VM / Sandbox Detection | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Timing-Based Evasion | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| API Hashing | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Process Injection Techniques | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Environment Variable / Hostname Checks | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Trojanized Plugin Analysis with Custom Alphabet C2 Decoding (INShAck 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| 脚本解混淆速查技巧族：JS、PowerShell 与编码载荷 | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Obfuscated Scripts | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| JavaScript & PowerShell Deobfuscation | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

## 常见陷阱

- 只按关键词跳页，没有先构造最小证据。
- 照搬 raw 中的一次性 payload，没有检查当前题的边界条件。
- 忽略相邻技巧之间的 pivot，导致在错误方向上继续投入时间。

## 关联技巧

- [powershell-staged-payload-and-clipboard-phishing.md](powershell-staged-payload-and-clipboard-phishing.md)
- [malware-c2-session-key-and-protocol-recovery.md](malware-c2-session-key-and-protocol-recovery.md)
- [pe-and-dotnet.md](pe-and-dotnet.md)
- [malware-tooling.md](malware-tooling.md)

## 原始资料

- [scripts-and-obfuscation.md](../raw/malware/scripts-and-obfuscation.md)
