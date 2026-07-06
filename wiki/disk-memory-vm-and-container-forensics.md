---
type: family
tags: [forensics, family, disk, memory, vm, container, cloud]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/disk-memory-vm-and-container-forensics.md
updated: 2026-06-12
---

# Disk, Memory, VM and Container Forensics

## 作用边界

本页是磁盘、内存、虚拟机、容器和云存储取证 family。它负责判断证据源是完整镜像、内存快照、VM/快照、core dump、Android 备份、Docker layer、云 bucket、BSON/数据库碎片还是勒索脚本残留。

它与相邻页的区别：本页偏载体入口和取证流程选择；[filesystems-memory-dumps-and-raid.md](filesystems-memory-dumps-and-raid.md) 偏底层文件系统、RAID、minidump、VMDK sparse 和具体恢复模式；[filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md) 偏进入文件/归档层之后的修复、删除恢复、ZipCrypto 和结构化文件恢复。

## 识别信号

- 附件是 `.raw`、`.img`、`.dd`、`.vmdk`、`.ova`、`.vmem`、`.vmss`、core dump、Android backup、Docker image/layer、KAPE collection 或云存储导出。
- flag 藏在历史状态、已删除文件、环境变量、进程内存、容器层、构建命令、数据库碎片或勒索脚本 key 中。
- 需要只读挂载、carving、Volatility/MemProcFS、layer diff、timeline、strings、registry/Amcache/MFT 或数据库解析。

## 最小证据

- 识别容器格式和层级：磁盘分区、文件系统、VM 容器、内存类型、Docker layer、Android backup 或云对象。
- 记录只读处理方式和导出的中间文件，避免污染证据。
- 明确搜索目标：文件内容、凭据、key、进程、环境变量、网络痕迹、时间线或删除历史。
- 能解释从载体到最终 artifact 的恢复路径。

## 首轮路由

| 证据形态 | 首轮判断 | 下一跳 |
|---|---|---|
| 完整磁盘镜像、分区表、删除文件 | 先只读挂载和分区识别，再决定 filesystem/recovery 工具 | [filesystems-memory-dumps-and-raid.md](filesystems-memory-dumps-and-raid.md) |
| 可挂载文件系统内的删除文件、损坏压缩包、ZipCrypto | 载体已经清楚，重点是文件/归档层恢复 | [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md) |
| 内存 dump、VM snapshot、core dump | 先判断 Volatility profile/OS/进程，再做 strings、filescan、mft、环境变量或进程提取 | [windows-registry-logs-and-credentials.md](windows-registry-logs-and-credentials.md) |
| OVA/VMDK/VMDK sparse、VMware snapshot | 先展开容器并识别磁盘/内存组件，不要只看外层 tar | [filesystems-memory-dumps-and-raid.md](filesystems-memory-dumps-and-raid.md) |
| Docker image/layer、container diff、history | 先看 layer.tar、whiteout、history、ENV/ARG 和删除文件 | [linux-git-browser-and-container-forensics.md](linux-git-browser-and-container-forensics.md) |
| Android backup/APK/data 分区 | 先分离 APK、shared_prefs、SQLite、WiFi/浏览器/应用数据 | [mobile-firmware-kernel-and-game-re.md](mobile-firmware-kernel-and-game-re.md) |
| 云 bucket/versioning 导出 | 先查公开权限、版本历史、对象元数据和删除版本 | [osint-account-public-media-correlation.md](osint-account-public-media-correlation.md) |
| PowerShell 勒索/恶意脚本内存残留 | 先提取脚本和 key，再转 malware 或 crypto 解密 | [scripts-and-obfuscation.md](scripts-and-obfuscation.md), [rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md) |

## 合并与拆分结论

本页保留为 family，不与 [filesystems-memory-dumps-and-raid.md](filesystems-memory-dumps-and-raid.md) 合并：本页负责选择取证载体和流程，后者负责具体底层恢复。若后续页面过长，可按“内存/VM”和“容器/云”再拆。

## 常见陷阱

- 直接挂载写入镜像，破坏时间戳或恢复状态。
- 只跑 strings，没先识别文件系统和快照层级。
- Docker 题只看最终层，忽略历史层和 whiteout 删除痕迹。
- 内存题没确认 OS/profile，插件输出误导。
- 云存储题只看当前对象，没查 versioning 和 metadata。

## 关联技巧

- [cross-domain-forensics-technique-map.md](cross-domain-forensics-technique-map.md)
- [filesystems-memory-dumps-and-raid.md](filesystems-memory-dumps-and-raid.md)
- [linux-git-browser-and-container-forensics.md](linux-git-browser-and-container-forensics.md)
- [windows-registry-logs-and-credentials.md](windows-registry-logs-and-credentials.md)
- [forensics-tooling.md](forensics-tooling.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [D3CTF2019-vera-wp](../raw/misc/D3CTF2019-vera-wp.md) | VeraCrypt 容器和图像光栅恢复组合，先用题面线索找密码再处理解出的媒体 artifact。 |
| [D3CTF2021-virtual-love-revenge-2-0-wp](../raw/misc/D3CTF2021-virtual-love-revenge-2-0-wp.md) | VMware 配置损坏、零宽字符字典和单用户模式取证组合，先修复虚拟机再进入磁盘证据链。 |
| [D3CTF2022-wannawacca-wp](../raw/misc/D3CTF2022-wannawacca-wp.md) | Windows 内存镜像、勒索样本和流量协议复现组合，先 dump 可疑进程和文件再还原加密链。 |
| [D3CTF2019-disappeared-memory-wp](../raw/reverse/D3CTF2019-disappeared-memory-wp.md) | Windows 10 compressed memory 导致关键页缺失，先从 dump/PTE/Store Manager 恢复数据页。 |

## 原始资料

- [disk-memory-vm-and-container-forensics.md](../raw/forensics/disk-memory-vm-and-container-forensics.md)
