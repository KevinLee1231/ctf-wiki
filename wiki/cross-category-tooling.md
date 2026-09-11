---
type: tooling
tags: [cross-category, tooling, triage, tools, environment]
skills: [ctf-solve-challenge]
---

# Cross-Category Triage Tooling

本页只服务类别尚未确定时的低成本首检。出现决定性证据后进入正式专项 skill，并读取该方向唯一的 `*-tooling.md`；这里不维护专项工具合集。

## 平台与调用

Windows 是默认工作平台。首检脚本使用 `D:/文档/新建文件夹/venv/Scripts/python.exe`（Python 3.14.3），不需要为编码、文件头或哈希检查进入 WSL。WSL `ctf-tools` 只承担 Linux 原生工作所需的工具及必要依赖，不能从环境名推断某个通用分析包仍在其中。

所有终端示例从 `pwsh` 发起。Windows 路径如 `D:/题目路径/unknown.bin`，传给 WSL 时对应 `/mnt/d/题目路径/unknown.bin`。正文中的题目路径都是需要替换的输入占位。

```pwsh
Get-Item -LiteralPath "D:/题目路径/unknown.bin" | Select-Object Name,Length
Get-FileHash -LiteralPath "D:/题目路径/unknown.bin" -Algorithm SHA256
& "D:/文档/新建文件夹/venv/Scripts/python.exe" -c 'from pathlib import Path; p=Path("D:/题目路径/unknown.bin"); print(p.open("rb").read(64).hex(" "))'
```

已有 WSL 格式识别工具可用于补充容器或架构信息；它们是 APT 命令，不套 Conda：

```pwsh
wsl -d kali-linux --exec /usr/bin/file "/mnt/d/题目路径/unknown.bin"
wsl -d kali-linux --exec /usr/bin/xxd -l 128 -g 1 "/mnt/d/题目路径/unknown.bin"
wsl -d kali-linux --exec /usr/bin/strings -n 8 "/mnt/d/题目路径/unknown.bin"
```

| 入口 | 当前状态 | 首检用途 |
|---|---|---|
| PowerShell 文件属性、SHA-256 | Windows 内置命令可用 | 固定文件大小和身份，避免分析错附件 |
| Python 标准库 | 上述 Windows venv | 用 `pathlib`、`struct`、`base64`、`json`、`zipfile` 检查少量字节或已知表示层 |
| `/usr/bin/file` | APT `file`：`1:5.47-4` | 识别容器、架构和文件类型线索 |
| `/usr/bin/strings` | APT `binutils`：`2.47-2` | 提取可打印字符串；字符串是线索，不是格式证明 |
| `/usr/bin/xxd` | 已有系统入口 | 有界十六进制查看；不默认转储整个大文件 |

## 首检后的选择

| 首检证据 | 下一步 |
|---|---|
| Base/hex/URL、码表、多层可逆编码 | [crypto-tooling.md](crypto-tooling.md) 与 [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) |
| Native、字节码、VM、程序比较逻辑 | [reverse-tooling.md](reverse-tooling.md)；已确认利用原语则进 [pwn-tooling.md](pwn-tooling.md) |
| 镜像、PCAP、文档或文件容器 | [forensics-tooling.md](forensics-tooling.md) |
| 像素、QR 碎片、音频或隐藏载荷 | [stego-tooling.md](stego-tooling.md) |
| HTTP、认证态或服务端输入边界 | [web-tooling.md](web-tooling.md) |
| 仍有跨方向冲突 | [cross-category-triage-family.md](cross-category-triage-family.md)，继续补一个能区分假设的证据 |

## 失败处理与环境边界

- 格式识别失败时保留原始字节，结合文件头、尾部、长度与容器结构核对；异常格式可转 [exotic-encodings-and-file-formats.md](exotic-encodings-and-file-formats.md)，不要直接批量解码或爆破。
- Jail 或 sandbox 只有部分输出时，记录允许语法、builtin、fd、错误回显和过滤阶段，再转 [pyjails.md](pyjails.md)、[bashjails.md](bashjails.md) 或 [interactive-containers-jails-and-solvers.md](interactive-containers-jails-and-solvers.md)。
- Z3、图像、模型、反编译和网络工具由对应专项页选择；它们不是未知题目的默认依赖。约束反馈明确后可查 [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md)。
- 缺包先评估 Windows venv 中兼容的新版本或已有替代方案。只有任务确需 Linux 能力才考虑 WSL；WSL 新增、升级、重装均须按全局 AGENTS.md 说明用途、平台必要性、目标和影响并取得明确同意。
