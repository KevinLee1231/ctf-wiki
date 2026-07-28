---
type: technique
tags: [forensics, archive, zip, structure-repair, carving]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/disk-recovery.md
  - ../raw/forensics/file-triage-archives-and-one-liners.md
  - ../raw/forensics/0xGame2024-week1-加密的压缩包-wp.md
  - ../raw/forensics/D3CTF2019-find-me-wp.md
updated: 2026-07-28
---

# Archive Structure Repair and Stream Carving

## 适用场景

ZIP、7z、tar、gzip 或嵌入式归档的 header、central directory、长度、flag、尾记录或分卷结构被破坏，但 local header、压缩流、CRC 或相邻文件边界仍足以恢复成员。

## 识别信号

- 解压器报告 central directory、offset、CRC、truncated stream 或 encryption flag 不一致。
- 文件尾/宿主文件内部仍能找到 `PK\x03\x04`、`PK\x05\x06` 或其它成员魔数。
- 本地文件头和中央目录对同一成员给出不同 flag、大小或压缩方法。
- 分卷或拼接文件中存在连续可验证的压缩流。

## 最小证据

- 保存原文件 hash、总长度、所有候选 header offset 和解压器原始错误。
- 明确修改的是哪个结构字段及依据，不用“能解压”代替结构解释。
- 修复后逐成员验证 CRC、文件签名、长度与容器边界。

## 解法骨架

1. 解析 local header、central directory、end record 与成员压缩方法。
2. 对比重复字段，确定被清零、错位或截断的 flag、长度和 offset。
3. 中央目录不可恢复时，从 local header 和压缩流边界逐成员 carve，再重建目录。
4. 对宿主文件先确认外层真实 EOF，再提取其后的内嵌归档。
5. 修复后递归识别成员格式，并保留从原 offset 到恢复文件的映射。

## 关键变体

- Central directory/end record 重建。
- General-purpose bit flag 或压缩方法字段伪装。
- Truncated/拼接压缩流和分卷恢复。
- 图片、文档或固件尾部的嵌入归档 carving。

## 常见陷阱

- 为通过解压器随意改 CRC/长度，掩盖真实数据损坏。
- 只修 local header，忘记同步中央目录中的对应字段。
- 把 ZipCrypto 密钥恢复混入结构修复；已知明文攻击有独立前提。
- 直接解压不可信路径穿越或 symlink 成员。

## 关联技巧

- [zipcrypto-known-plaintext-recovery.md](zipcrypto-known-plaintext-recovery.md)
- [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md)
- [file-format-and-embedded-payload-identification.md](file-format-and-embedded-payload-identification.md)

## 原始资料

- [disk-recovery.md](../raw/forensics/disk-recovery.md)
- [file-triage-archives-and-one-liners.md](../raw/forensics/file-triage-archives-and-one-liners.md)
- [0xGame2024-week1-加密的压缩包-wp](../raw/forensics/0xGame2024-week1-加密的压缩包-wp.md)
- [D3CTF2019-find-me-wp](../raw/forensics/D3CTF2019-find-me-wp.md)
