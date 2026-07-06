---
type: family
tags: [forensics, family, linux, git, browser, container, credentials]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/linux-git-browser-and-container-forensics.md
  - ../raw/misc/WMCTF2025-githacker-wp.md
  - ../raw/misc/ACTF2026-ezssh-wp.md
updated: 2026-07-06
---

# Linux, Git, Browser and Container Forensics

## 作用边界

本页是 Linux/Git/Browser/Container 证据源 family，用于从 Linux 日志、Git 对象库、浏览器 profile、Docker layer/history、KeePass、容器镜像和运行中脚本/进程中恢复历史状态、凭据或被删除数据。

它不替代磁盘/内存入口页，也不替代 Pentest 横向移动页。当前任务仍是“从已有 artifacts 恢复证据”时用本页；如果已拿到可用凭据并开始登录、隧道或提权，转到 [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md) 或 [linux-privesc.md](linux-privesc.md)。

## 识别信号

- 附件包含 `.git/`、Docker image tar、`manifest.json`、`blobs/sha256`、Linux `/var/log`、shell history、Chrome/Firefox profile、KeePass `.kdbx`、deleted source、Python 进程或浏览器 SQLite。
- 题面提示 rollback、squash、history、cache、container layer、Docker build secret、browser password、Local Storage、KeePass、bash history、deleted file but process still running。
- 需要恢复“过去存在过”的内容：Git dangling object、Docker build history、浏览器历史/凭据、日志中的攻击链、进程内存中的 Python source。

## 最小证据

- 明确 artifact 所属层：文件系统日志、Git object database、浏览器 profile、Docker layer/config、数据库/密码库、运行中进程。
- 保留原始导出和恢复对象：`git show` 输出、Docker config JSON、SQLite 查询结果、解密后的浏览器凭据、KeePass hash、进程 dump。
- 对凭据类发现，要区分“可恢复”与“可使用”：是否需要 master key、wordlist、DPAPI、key4.db、KeePass 口令或上下文词表。
- 对 Git/容器历史，不能只看当前文件树；要检查 reflog、fsck、dangling objects、layer tar 和 config history。

## 首轮路由

| 证据形态 | 先做什么 | 下一跳 |
|---|---|---|
| `.git` 暴露、历史被 squash/rebase/rollback | 先查 `git reflog --all`、`git fsck --unreachable`、dangling commit/blob，再从对象库取历史版本。 | [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md) |
| Git blob 损坏 | 用 expected SHA-1 和 `git hash-object` 判断是单字节修复、上下文修复还是对象缺失。 | [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md) |
| Docker image/layer | 先读 `manifest.json` 和 config blob 的 `history[].created_by`，再解 layer tar；删除不代表历史命令消失。 | [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md) |
| Chrome/Edge/Firefox profile | 查询 History/Downloads/Cookies/Login Data/Local Storage/session restore；密码解密先找 Local State、DPAPI/keychain、Firefox `key4.db`。 | [windows-registry-logs-and-credentials.md](windows-registry-logs-and-credentials.md) |
| Linux auth/log/bash history | 先按时间线拼 SSH、sudo、cron、下载、执行、网络连接，再和 PCAP 或文件落点交叉验证。 | [network-covert-auth-and-reassembly.md](network-covert-auth-and-reassembly.md) |
| KeePass 或本地密码库 | 先识别 KDBX 版本和 KDF；KeePass v4/Argon2 需要合适的 john/hashcat/专用工具和上下文词表。 | [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md) |
| Python 源码删除但进程仍在 | 先确认 PID 和 ptrace 条件，再从 code object/global variable 恢复源码或 secret。 | [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md) |
| PCAP 中带 USB/TFTP/TLS 资料 | 先按协议导出，若得到文件/凭据再回到本页或 PCAP family。 | [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md) |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 | 对应路线 |
|---|---|---|
| [WMCTF2025-githacker-wp](../raw/misc/WMCTF2025-githacker-wp.md) | `git log` 被回滚隐藏时，`git reflog` 可恢复关键提交、密码文件和被删除容器；后续还可能需要按文件格式继续恢复。 | Git reflog + dangling object + 后续文件识别 |
| [ACTF2026-ezssh-wp](../raw/misc/ACTF2026-ezssh-wp.md) | SSH 横向链中 `.bash_history`、`authorized_keys`、Debian 弱 key、Git dangling commit 和 systemd backup unit 各提供一段凭据或配置。 | SSH/Git/backup artifact 串联 |

## 合并与拆分结论

- 保留为 `family`：raw 覆盖 Linux 日志、Git、Docker、浏览器、KeePass、Python 内存源码和部分协议导出，核心价值是证据源二级分流。
- 不并入 [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md)：后者负责磁盘/内存/VM/容器入口，本页负责进入 Linux/Git/Browser/Container artifact 后的路线选择。
- 暂不拆 Git、Browser、Docker 三页：当前 raw 中它们都服务于“历史状态和凭据恢复”，且常在同一题链上相互跳转。

## 常见陷阱

- Git 只看 `git log`，忽略 reflog、stash、dangling object 和 `.git/objects`。
- Docker 只解最终 layer，没有读 config history，错过构建时泄露的 secret。
- 浏览器取证只查 History，不查 Downloads、Cookies、Login Data、Local Storage 和 session restore。
- KeePass v4 继续套旧 `keepass2john`，没有确认 KDF/Argon2 支持。
- 从日志中拿到凭据后直接进入渗透流程，没有记录凭据来源和可验证服务范围。

## 关联技巧

- [cross-domain-forensics-technique-map.md](cross-domain-forensics-technique-map.md)
- [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md)
- [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md)
- [windows-registry-logs-and-credentials.md](windows-registry-logs-and-credentials.md)
- [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md)
- [network-covert-auth-and-reassembly.md](network-covert-auth-and-reassembly.md)
- [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md)
- [forensics-tooling.md](forensics-tooling.md)

## 原始资料

- [linux-git-browser-and-container-forensics.md](../raw/forensics/linux-git-browser-and-container-forensics.md)
- [WMCTF2025-githacker-wp](../raw/misc/WMCTF2025-githacker-wp.md)
- [ACTF2026-ezssh-wp](../raw/misc/ACTF2026-ezssh-wp.md)
