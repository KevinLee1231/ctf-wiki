---
type: technique
tags: [forensics, filesystem, deleted-file, metadata, carving]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/disk-recovery.md
  - ../raw/forensics/filesystems-memory-dumps-and-raid.md
updated: 2026-07-27
---

# Filesystem Metadata and Deleted-Artifact Recovery

## 适用场景

磁盘/分区/文件系统镜像中目标文件已删除、重命名或目录项损坏，但 inode/MFT、journal、未分配块、文件签名或历史 metadata 仍可恢复内容和时间线。

## 识别信号

- 镜像包含 NTFS/ext/FAT/APFS 等文件系统而非单一文件。
- 目录缺少目标，但 MFT/inode/journal 或 unallocated space 有残留。
- 文件名、大小、时间、块地址或 signature 可作为恢复锚点。

## 最小证据

- 对原镜像只读操作并记录 hash、分区 offset 和 sector size。
- 确认文件系统类型、allocation 状态和 metadata 记录。
- 恢复文件需记录来源块/记录号并验证 magic/结构。

## 解法骨架

1. 解析分区表和文件系统，建立目录/MFT/inode 时间线。
2. 优先按 metadata 恢复完整 extents；metadata 不足再做 carving。
3. 用 journal、USN、目录索引和相邻文件关联名称与时间。
4. 对恢复对象计算 hash并递归解析内部格式。

## 关键变体

- NTFS MFT/USN/ADS。
- ext inode/journal 与已删除目录项。
- Raw carving 与碎片化文件重组。

## 常见陷阱

- 直接挂载可写，改变证据 metadata。
- 只用 carving，丢失文件名和时间上下文。
- 拼接碎片仅靠 magic，没有结构校验。

## 关联技巧

- [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md)
- [filesystems-memory-dumps-and-raid.md](filesystems-memory-dumps-and-raid.md)
- [raid-vm-disk-and-fragmented-volume-reconstruction.md](raid-vm-disk-and-fragmented-volume-reconstruction.md)

## 原始资料

- [disk-recovery.md](../raw/forensics/disk-recovery.md)
- [filesystems-memory-dumps-and-raid.md](../raw/forensics/filesystems-memory-dumps-and-raid.md)
