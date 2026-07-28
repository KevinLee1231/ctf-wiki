---
type: technique
tags: [forensics, raid, vmdk, disk, volume, reconstruction]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/filesystems-memory-dumps-and-raid.md
  - ../raw/forensics/disk-memory-vm-and-container-forensics.md
  - ../raw/forensics/D3CTF2021-virtual-love-revenge-2-0-wp.md
updated: 2026-07-28
---

# RAID, VM-Disk and Fragmented-Volume Reconstruction

## 适用场景

证据由多块 RAID 成员、VMDK sparse extent、分卷、损坏 partition 或碎片化块组成；必须先恢复正确几何、顺序和映射，之后文件系统工具才有意义。

## 识别信号

- 多个等长镜像、stripe/parity 模式或 RAID metadata。
- VMDK descriptor、grain table/extent 缺失或分离。
- Partition table 损坏，但文件系统 superblock/boot sector 可定位。

## 最小证据

- 记录每个成员 hash、大小、sector/stripe/grain 参数。
- 用文件系统 magic、已知文件跨 stripe 连续性验证排列。
- 重建镜像的分区和文件系统检查结果可重复。

## 解法骨架

1. 解析 metadata/descriptor；缺失时枚举小范围顺序、stripe 与 parity。
2. 用 magic、superblock 和跨块结构为候选打分。
3. 生成只读虚拟映射或新镜像，不改原成员。
4. 在重建层上执行文件系统 metadata 恢复和 carving。

## 关键变体

- RAID0/5/6 顺序、stripe 和 parity。
- VMDK sparse grain mapping。
- 损坏分区表/分卷与碎片化文件。

## 常见陷阱

- 只因 mount 成功就认定几何正确。
- 将成员原地拼接/修复，破坏原始证据。
- 忽略 sector 与 stripe 单位换算。

## 关联技巧

- [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md)
- [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md)
- [filesystem-metadata-and-deleted-artifact-recovery.md](filesystem-metadata-and-deleted-artifact-recovery.md)

## 原始资料

- [filesystems-memory-dumps-and-raid.md](../raw/forensics/filesystems-memory-dumps-and-raid.md)
- [disk-memory-vm-and-container-forensics.md](../raw/forensics/disk-memory-vm-and-container-forensics.md)
- [D3CTF2021-virtual-love-revenge-2-0-wp](../raw/forensics/D3CTF2021-virtual-love-revenge-2-0-wp.md)
