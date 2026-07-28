---
type: technique
tags: [malware, reverse, pe, dotnet, config, resource]
skills: [ctf-malware, ctf-reverse]
raw:
  - ../raw/malware/pe-and-dotnet.md
  - ../raw/malware/VNCTF2026-vnshell-wp.md
updated: 2026-07-28
---

# PE/.NET Config and Resource Extraction

## 适用场景

PE/.NET 恶意样本的核心目标是恢复嵌入资源、配置、C2、campaign key 或二阶段载荷，而非构造内存利用；需要结合静态 metadata 与运行时解密点。

## 识别信号

- `.rsrc`、overlay、managed resources、加密 blob 或大字节数组。
- 初始化函数解密域名、key、mutex、campaign ID。
- PyInstaller/.NET single-file/packer 包含可提取内嵌模块。

## 最小证据

- 记录样本 hash、架构、managed/native、节区和资源清单。
- 定位配置读取/解密函数及其输入 key/IV。
- 提取结果有结构、域名/IP、magic 或运行时使用证据。

## 解法骨架

1. 解析 PE headers、imports、resources、overlay 和 CLR metadata。
2. 搜索配置访问点和字符串/资源解密循环。
3. 优先离线重写解密器；必要时在解密后、使用前 dump。
4. 将 C2/config/载荷与行为 trace 交叉验证。

## 关键变体

- Native PE resource/overlay。
- .NET managed resource 与反射加载。
- PyInstaller/PyArmor/单文件打包的嵌入模块。

## 常见陷阱

- 看到高熵节区就直接判定为壳。
- 字符串解密结果未证明被程序使用。
- 只提取主程序，漏掉资源中的二阶段。

## 关联技巧

- [pe-and-dotnet.md](pe-and-dotnet.md)
- [staged-loader-and-runtime-image-recovery.md](staged-loader-and-runtime-image-recovery.md)
- [malware-c2-session-key-and-protocol-recovery.md](malware-c2-session-key-and-protocol-recovery.md)

## 原始资料

- [pe-and-dotnet.md](../raw/malware/pe-and-dotnet.md)
- [VNCTF2026-vnshell-wp](../raw/malware/VNCTF2026-vnshell-wp.md)
