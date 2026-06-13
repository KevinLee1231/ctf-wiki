---
type: family
tags: [malware, family, pe, dotnet, config, shellcode]
skills: [ctf-malware, ctf-reverse, ctf-forensics]
raw:
  - ../raw/malware/pe-and-dotnet.md
updated: 2026-06-12
---

# PE, .NET, and Binary Malware Analysis

## 作用边界

本页是 PE/.NET 恶意样本 family，覆盖 PE 静态分析、sandbox evasion、配置提取、.NET DNS C2、PyInstaller/PyArmor、插件化样本、YARA、shellcode 和内存取证联动。它不是 PE 工具百科，重点是从样本行为判断先拆哪一层、提取什么配置、是否需要隔离动态验证。

## 识别信号

- 样本是 PE/.NET、PyInstaller、shellcode、插件、loader 或被混淆的二进制载荷。
- 存在 C2、DNS 查询、加密配置 blob、反调试/反沙箱、分阶段 payload、YARA/IOC 或内存残留。
- 目标是提取 flag、配置、会话 key、C2 协议、payload 或恶意行为链。

## 最小证据

- 文件类型、架构、packer/混淆、导入、入口、字符串和是否 .NET。
- 配置来源：resource、section、registry、环境变量、DNS、命令行、内存或网络流量。
- 动态验证必须在隔离环境，并明确要观察的行为：解密、C2、文件释放、进程注入或 shellcode。
- 输出 IOC/config 时保留偏移、解密 key 和复现脚本/命令。

## 路由表

| 证据 | 先验证 | 下一跳 |
|---|---|---|
| 普通 PE | section、import、resource、TLS callback 和 packer | 静态拆层或轻量动态 |
| .NET 样本 | dnlib/ILSpy、混淆器、资源和 C2 类 | 还原 IL、解密 string/config |
| DNS-based C2 | 查询域名、TXT/A 记录、编码和轮询逻辑 | [dns.md](dns.md) 或 network forensics |
| PyInstaller/PyArmor | pyz、marshal、runtime hook 和 obfuscated pyc | 转 reverse Python 页 |
| shellcode | 架构、入口、API hash、解密循环和 syscall | 转 reverse/pwn runtime |
| memory forensics | 进程 dump、injected region、config in memory | [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md) |
| YARA/IOC | 匹配规则是否解释真实行为 | 输出规则时保留字段来源 |

## 合并与拆分结论

- 保留为 family：PE/.NET 恶意样本通常是拆层、配置、C2、shellcode和内存证据组合。
- 不合并进 `malware-c2-session-key-and-protocol-recovery.md`：该页聚焦通信和密钥恢复，本页负责样本本体分析入口。
- 不合并进 reverse PE 页：恶意样本还需要 IOC、C2 和沙箱边界。

## 常见误判

- 直接运行样本而没有明确观察目标和隔离边界。
- 只提 strings，不还原解密函数和配置结构。
- .NET 样本只反编译不处理资源和混淆器初始化。
- YARA 规则只匹配壳/库代码，不能说明恶意行为。

## 关联页面

- [malware-c2-session-key-and-protocol-recovery.md](malware-c2-session-key-and-protocol-recovery.md)
- [powershell-staged-payload-and-clipboard-phishing.md](powershell-staged-payload-and-clipboard-phishing.md)
- [scripts-and-obfuscation.md](scripts-and-obfuscation.md)
- [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md)
- [malware-tooling.md](malware-tooling.md)

## 原始资料

- [pe-and-dotnet.md](../raw/malware/pe-and-dotnet.md)
