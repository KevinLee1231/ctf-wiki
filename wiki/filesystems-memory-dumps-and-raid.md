---
type: family
tags: [forensics, family, filesystem, memory-dump, raid, partition]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/filesystems-memory-dumps-and-raid.md
updated: 2026-06-12
---

# Filesystems, Memory Dumps and RAID

## 作用边界

本页是底层存储和内存恢复 family，覆盖删除分区、ZFS/APFS/HFS+/GPT、VMDK sparse、minidump/core dump、memory key carving、RAID5 XOR、TrueCrypt/VeraCrypt、SQLite/Kyoto Cabinet 历史、BSON 重组和压缩 blob 检测。

它不负责选择取证载体；载体入口先看 [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md)。本页负责已经确认要做文件系统、分区、内存 dump 或 RAID 级恢复后的具体路由。

## 共同识别信号

- 文件系统结构损坏、分区表缺失、GUID/元数据异常、快照/版本历史、RAID 成员缺失、VMDK sparse 或 minidump 字符串可见。
- 明文不在当前目录树，而在已删除分区、快照、resource fork、resident MFT、内存环境变量、数据库 diff 或 RAID parity 中。
- 需要按结构恢复，而不是简单 `strings` 或 binwalk。

## 最小证据

- 记录镜像大小、扇区大小、分区表、文件系统类型、块大小、成员盘顺序或 dump 类型。
- 对恢复类题，先只读复制并保留工具输出。
- 对内存 key，记录进程、地址附近上下文、算法线索和解密验证。
- 对 RAID/压缩/数据库，记录重组顺序、校验和以及恢复后的文件签名。

## 首轮路由

| 证据形态 | 首轮判断 | 下一跳 |
|---|---|---|
| 分区删除、GPT GUID 藏数据、ZFS/APFS/HFS+ 快照 | 先恢复分区/快照/元数据，再挂载或导出历史版本 | [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md) |
| VMDK sparse、OVA、VM snapshot | 先识别 sparse grain 和 snapshot 链，再导出完整磁盘或内存 | [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md) |
| Windows minidump、core dump、memory string carving | 先判断 dump 类型和进程，再提取环境变量、key、路径和内存文件 | [windows-registry-logs-and-credentials.md](windows-registry-logs-and-credentials.md) |
| 勒索脚本/恶意样本 key 在内存中 | 先提取 key 和算法，再转 malware/crypto 解密 | [scripts-and-obfuscation.md](scripts-and-obfuscation.md), [rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md) |
| RAID5/XOR、缺盘恢复 | 先确定成员顺序、chunk size、parity 方向，再重建文件系统 | [forensics-tooling.md](forensics-tooling.md) |
| TrueCrypt/VeraCrypt 高熵卷或 keyfile | 先确认卷大小和上下文线索，再尝试密码/keyfile/隐藏卷 | [file-triage-archives-and-one-liners.md](file-triage-archives-and-one-liners.md) |
| SQLite/Kyoto Cabinet/BSON/diff history | 先解析记录结构和顺序，再重建历史内容 | [linux-git-browser-and-container-forensics.md](linux-git-browser-and-container-forensics.md) |

## 合并与拆分结论

本页保留为 family，并与 [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md) 分工：前者是底层恢复路线，后者是载体选择入口。当前不拆分 RAID、minidump、ZFS、TrueCrypt 等小页，因为 raw 仍以短案例集合为主。

## 常见陷阱

- 未确认扇区/块大小就恢复分区或 RAID。
- 对内存 dump 只跑 strings，没利用进程、环境变量和文件映射上下文。
- VMDK sparse 没按 grain table 解析，直接当 raw 镜像挂载。
- TrueCrypt/VeraCrypt 高熵文件没结合上下文寻找 keyfile。
- SQLite/diff/数据库历史不按版本顺序重放。

## 关联技巧

- [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md)
- [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md)
- [windows-registry-logs-and-credentials.md](windows-registry-logs-and-credentials.md)
- [linux-git-browser-and-container-forensics.md](linux-git-browser-and-container-forensics.md)
- [forensics-tooling.md](forensics-tooling.md)

## 原始资料

- [filesystems-memory-dumps-and-raid.md](../raw/forensics/filesystems-memory-dumps-and-raid.md)
