---
type: technique
tags: [forensics, archive, repair, zipcrypto, known-plaintext]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/disk-recovery.md
  - ../raw/forensics/file-triage-archives-and-one-liners.md
updated: 2026-07-27
---

# Archive Repair and Known-Plaintext Recovery

## 适用场景

ZIP/7z/tar/gzip 等归档的 header/central directory 损坏、分卷缺失或采用 ZipCrypto 等弱加密；可从 local header、CRC、已知文件和压缩流边界恢复。

## 识别信号

- 解压器报告 central directory、CRC、offset 或 truncated stream 错误。
- Local file header 仍存在，或归档尾部/分卷可找回。
- ZipCrypto 成员有已知/可取得相同压缩明文。

## 最小证据

- 保存原归档 hash、长度、所有 header offset 和错误位置。
- 区分明文字节相同与压缩流相同。
- 修复后每个成员通过 CRC/格式校验。

## 解法骨架

1. 解析 local/central/end records 与成员压缩方法。
2. 从 local header 重建目录、长度和 offset，必要时逐流 carve。
3. ZipCrypto 已知明文先确认压缩流、CRC 和大小完全匹配，再恢复 key。
4. 解密/修复后递归检查内嵌归档和路径安全。

## 关键变体

- Central directory 重建。
- Truncated/拼接压缩流修复。
- ZipCrypto known-plaintext 与 CRC 辅助。

## 常见陷阱

- 只有相同图片视觉内容，不代表 deflate 流相同。
- 为通过解压器随意改 CRC/长度，掩盖数据损坏。
- 直接解压不可信路径穿越/symlink 成员。

## 关联技巧

- [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md)
- [file-triage-archives-and-one-liners.md](file-triage-archives-and-one-liners.md)
- [file-format-and-embedded-payload-identification.md](file-format-and-embedded-payload-identification.md)

## 原始资料

- [disk-recovery.md](../raw/forensics/disk-recovery.md)
- [file-triage-archives-and-one-liners.md](../raw/forensics/file-triage-archives-and-one-liners.md)
