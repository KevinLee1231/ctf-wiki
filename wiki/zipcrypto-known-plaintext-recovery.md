---
type: technique
tags: [forensics, archive, zipcrypto, known-plaintext, bkcrack]
skills: [ctf-forensics, ctf-crypto]
raw:
  - ../raw/forensics/file-triage-archives-and-one-liners.md
  - ../raw/forensics/D3CTF2021-robust-wp.md
updated: 2026-07-28
---

# ZipCrypto Known-Plaintext Recovery

## 适用场景

ZIP 成员采用传统 ZipCrypto，且能取得同一成员的一段精确明文或等价文件；目标是恢复内部密钥状态、解密成员或生成可被原归档接受的新密文，而不是爆破用户密码。

## 识别信号

- ZIP general-purpose bit flag 表明传统加密，且不是 AES extra field。
- 已知文件名、模板、歌词、协议头或公开文件可能与加密成员内容相同。
- 成员压缩方法、未压缩大小、CRC 和可取得样本可对齐。
- `bkcrack`/pkcrack 报告已知字节数量或 offset 对齐问题。

## 最小证据

- 确认攻击输入是加密流对应的精确明文字节；若成员先 Deflate，通常需要匹配压缩后流。
- 记录成员名、压缩方法、CRC、大小、明文 offset 与密文 offset。
- 恢复出的内部 key 能解密完整成员并通过 CRC/文件格式校验。

## 解法骨架

1. 用 ZIP 解析器确认 ZipCrypto、成员顺序、data descriptor 和压缩方法。
2. 获取候选明文；必要时用完全相同参数重新压缩，比较长度和已知片段。
3. 向 `bkcrack` 提供原归档、成员名、明文文件及正确 offset，恢复三组内部 key。
4. 使用内部 key 解密目标成员或重建归档，再验证 CRC 和下层格式。
5. 如果候选只有语义相同而字节不一致，转模板对齐、更多已知头或密码线索，不能强行继续。

## 关键变体

- Stored 成员：明文字节可直接对齐。
- Deflated 成员：视觉/文本内容相同不代表压缩流相同。
- 部分已知明文：需要足够连续字节和准确 offset。
- 已恢复内部 key：可解密归档，不等同于恢复原用户密码。

## 常见陷阱

- 把现代 WinZip AES 当 ZipCrypto。
- 用“相同歌词/图片”的重新生成文件，却忽略编码、换行、metadata 或压缩参数差异。
- 把 ZIP 12 字节加密头计入错误 offset。
- 解出一段可读文本后不做完整 CRC 校验。

## 关联技巧

- [archive-structure-repair-and-stream-carving.md](archive-structure-repair-and-stream-carving.md)
- [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md)
- [file-triage-archives-and-one-liners.md](file-triage-archives-and-one-liners.md)

## 原始资料

- [file-triage-archives-and-one-liners.md](../raw/forensics/file-triage-archives-and-one-liners.md)
- [D3CTF2021-robust-wp](../raw/forensics/D3CTF2021-robust-wp.md)
