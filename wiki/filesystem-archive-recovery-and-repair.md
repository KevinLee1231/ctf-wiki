---
type: technique
tags: [forensics, filesystem, archive, recovery, known-plaintext, technique]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/disk-recovery.md
updated: 2026-06-11
---

# Filesystem and Archive Recovery Repair

## 适用场景

题目核心是从磁盘镜像、文件系统、压缩包、损坏容器、删除文件、嵌套归档或结构化文件中恢复数据。它不覆盖所有取证；内存/VM/容器运行态转到 [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md)，PCAP 转到 [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md)，图片/音频隐写转到对应媒体页。

## 识别信号

- 附件是 disk image、filesystem dump、ZIP/TAR/XZ/7z、VeraCrypt/LUKS、SQLite、损坏压缩包、删除文件或多层嵌套容器。
- `file`/magic/metadata 与后缀不一致，或存在 header、CRC、central directory、inode、superblock、partition table、snapshot/subvolume 异常。
- 解题依赖已知明文、文件头、结构字段、压缩字典、CRC、时间戳 seed 或文件系统元数据恢复。
- flag 可能不在当前可见文件，而在删除区、free space、历史版本、重复归档项、嵌套容器或损坏头修复后的内容中。

## 最小证据

- 保留原始附件只读副本，对每一步导出物记录来源偏移、文件名和命令。
- 先确认容器层级：分区、文件系统、压缩层、加密层、嵌套文件和最终 payload。
- 对损坏文件，至少定位一个可解释字段：magic、version、CRC、size、offset、inode、目录项或 central directory。
- 对 ZipCrypto/已知明文，先证明明文字段稳定且压缩方式匹配，再跑恢复工具。

## 解法骨架

1. 首检 magic、binwalk、7z/list、fsstat/fls、exiftool、strings 和熵，画出容器层级。
2. 若是文件系统，先查分区、目录项、删除文件、free space、snapshot/subvolume 和 orphan inode。
3. 若是压缩包，先修 header/size/CRC/central directory，再处理重复项、嵌套层或密码恢复。
4. 若可用已知明文，提取稳定文件头或结构字段，验证压缩方式后再用 bkcrack/同类工具恢复 key。
5. 每导出一层都重新做 magic/metadata 首检，直到拿到 flag、密钥或下一跳线索。

## 关键变体

| 变体 | 复用重点 |
|---|---|
| 文件系统删除恢复 | FAT/ext/XFS/BTRFS 的目录项、inode、free space、snapshot 和 orphan 机制不同，先用对应工具确认元数据仍在。 |
| 损坏压缩包修复 | ZIP/XZ/TAR 常见问题是 magic、size、CRC、central directory 或重复项；先修结构再谈爆破。 |
| ZipCrypto 已知明文 | 明文可来自 magic，也可来自 AVIF/ISOBMFF、PNG、PDF、DOCX 等结构字段；关键是压缩方法和明文偏移一致。 |
| 嵌套 Matryoshka 容器 | 每层都可能改变格式、压缩、编码或文件系统；不要一次性把所有导出物混在一起。 |
| 加密容器密钥恢复 | LUKS/VeraCrypt 等场景常需要内存 master key、题面密码线索或时间戳 seed。 |
| 结构化数据库恢复 | SQLite serial type、page header、WAL/journal 和字段类型可直接影响可见内容。 |

## 常见陷阱

- 在损坏归档上直接爆破密码，忽略 header/CRC/目录项本身已经坏了。
- 用 binwalk 一把梭导出后不记录偏移和层级，后续无法复现。
- 只按文件 magic 找已知明文，忽略 AVIF/ISOBMFF 这类稳定结构字段。
- 修改原始镜像而不留只读副本，导致恢复路径不可复查。

## 关联技巧

- [file-triage-archives-and-one-liners.md](file-triage-archives-and-one-liners.md)
- [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md)
- [filesystems-memory-dumps-and-raid.md](filesystems-memory-dumps-and-raid.md)
- [linux-git-browser-and-container-forensics.md](linux-git-browser-and-container-forensics.md)
- [cross-domain-forensics-technique-map.md](cross-domain-forensics-technique-map.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [SU_chaosWP](../raw/misc/SU_chaosWP.md) | ZipCrypto 已知明文不只用常见 magic；AVIF/ISOBMFF 头部字段也能提供稳定明文窗口。 |

## 原始资料

- [disk-recovery.md](../raw/forensics/disk-recovery.md)
