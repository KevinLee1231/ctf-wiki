---
type: technique
tags: [pentest, technique]
skills: [ctf-pentest]
raw:
  - ../raw/pentest/linux-privesc.md
updated: 2026-05-21
---

# Linux Privilege Escalation and Service Exploitation

## 适用场景

综合渗透、内网横向、凭据利用、隧道、AD/Kerberos 或提权链是主要障碍。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 目标给出多服务、多主机、凭据、SMB/WinRM/RDP/LDAP/Kerberos 或内网拓扑。
- 初始 foothold 后需要横向、端口转发、权限提升或凭据提取。
- flag 分布在不同权限层或内网服务中。
- 题面或 raw 线索能落到这些关键词之一：Sudo Wildcard Parameter Injection via fnmatch (Dump HTB)、Crafted Pcap for /etc/sudoers.d (Dump HTB)、Monit confcheck Process Command-Line Injection (Zero HTB)、Apache -d Last-Wins ServerRoot Override (Zero HTB)、Backup Cronjob SUID Abuse (Slonik HTB)、PostgreSQL COPY TO PROGRAM RCE (Slonik HTB)、PostgreSQL Backup Credential Extraction (Slonik HTB)、SSH Unix Socket Tunneling (Slonik HTB)。

## 最小证据

- 已完成主方向判断，并确认本页技巧比相邻技巧更能解释当前证据。
- 至少有一个可复现输入、输出、文件结构、数学关系、协议行为或运行时状态。
- 能指出 raw 案例中哪一个变体与当前题最接近，以及不同点在哪里。

## 解法骨架

1. 枚举服务、域信息和凭据面。
2. 验证凭据和访问边界。
3. 建立必要隧道或代理。
4. 按权限链逐步提权和横向。

## 关键变体

| 变体 | 复用重点 |
|---|---|
| Sudo Wildcard Parameter Injection via fnmatch (Dump HTB) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Crafted Pcap for /etc/sudoers.d (Dump HTB) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Monit confcheck Process Command-Line Injection (Zero HTB) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Apache -d Last-Wins ServerRoot Override (Zero HTB) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Backup Cronjob SUID Abuse (Slonik HTB) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| PostgreSQL COPY TO PROGRAM RCE (Slonik HTB) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| PostgreSQL Backup Credential Extraction (Slonik HTB) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| SSH Unix Socket Tunneling (Slonik HTB) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| NFS Share Exploitation for Sensitive Data (Slonik HTB) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| PaperCut Print Deploy Privilege Escalation (Bamboo HTB) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Squid Proxy Pivoting to Internal Services (Bamboo HTB) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Zabbix Admin Password Reset via MySQL (Watcher HTB) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| WinSSHTerm Encrypted Credential Decryption (Atlas HTB) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| sudo file -m Magic File Directory Traversal (OTW Advent 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| CVE-2018-19788 — polkit UID Integer Overflow → Systemd RCE (OTW Advent 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Sudo Glob Path + Symlink Confused Deputy via vim (STEM CTF 2019) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| TOCTOU Symlink Swap Race on FileChecker (STEM CTF 2019) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Linux 提权路径速查：SUID、Capabilities 与服务配置 | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| SUID Binary Exploitation | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Linux Privilege Escalation Quick Checks | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Docker Group Privilege Escalation | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| 现场提权证据速记 | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Additional Resources | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Useful One-Liners | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

## 常见陷阱

- 只按关键词跳页，没有先构造最小证据。
- 照搬 raw 中的一次性 payload，没有检查当前题的边界条件。
- 忽略相邻技巧之间的 pivot，导致在错误方向上继续投入时间。

## 关联技巧

- [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md)
- [pentest-tooling.md](pentest-tooling.md)
- [workflow-runner-internal-api-chain.md](workflow-runner-internal-api-chain.md)

## 原始资料

- [linux-privesc.md](../raw/pentest/linux-privesc.md)
