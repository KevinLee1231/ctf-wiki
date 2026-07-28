---
type: technique
tags: [pentest, privesc, linux, credentials, misconfiguration]
skills: [ctf-pentest]
raw:
  - ../raw/pentest/linux-privesc.md
  - ../raw/pentest/0xGame2025-week4-ezHack-wp.md
updated: 2026-07-28
---

# Local Privesc Misconfiguration and Credential Pivot

## 适用场景

已获得低权限 shell，提权依赖 sudo/SUID/capability、可写服务/定时任务、备份配置、进程环境或本机凭据，而非内核内存破坏。

## 识别信号

- `sudo -l`、SUID、file capability 或 service unit 暴露可控执行路径。
- 配置、备份、history、进程参数或环境变量包含凭据/token。
- root 任务会读取低权限可写脚本、路径、插件或配置。

## 最小证据

- 记录当前 UID/groups、主机/容器边界和可读写路径。
- 证明高权限执行点确实使用攻击者可控输入。
- 凭据必须在目标服务/用户上验证，避免无效历史秘密。

## 解法骨架

1. 先做低噪声枚举：身份、sudo、SUID/caps、服务、计划任务、挂载和进程。
2. 按“高权限执行者 -> 可控文件/参数 -> 可观察触发”建立攻击边。
3. 单独审计备份、配置、history、数据库和进程环境中的凭据。
4. 选择最小副作用路线，验证提权后再读取目标证据。

## 关键变体

- sudo/SUID/capability 功能滥用。
- 可写 service/timer/cron/plugin。
- 凭据复用、备份 secret 和本机服务 pivot。

## 常见陷阱

- 将容器 root 误当宿主 root。
- 盲跑枚举脚本，忽略可控到高权限触发的因果链。
- 修改系统配置后无法恢复或触发时机不可控。

## 关联技巧

- [linux-privesc.md](linux-privesc.md)
- [source-audit-hidden-backdoor-and-debug-mode-discovery.md](source-audit-hidden-backdoor-and-debug-mode-discovery.md)
- [restricted-shell-feature-and-output-channel-escape.md](restricted-shell-feature-and-output-channel-escape.md)

## 原始资料

- [linux-privesc.md](../raw/pentest/linux-privesc.md)
- [0xGame2025-week4-ezHack-wp](../raw/pentest/0xGame2025-week4-ezHack-wp.md)
