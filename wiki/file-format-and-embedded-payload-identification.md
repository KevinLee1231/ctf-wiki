---
type: technique
tags: [forensics, crypto, file-format, embedded-data, triage]
skills: [ctf-forensics, ctf-stego, ctf-crypto]
raw:
  - ../raw/crypto/encodings-qr-and-esolangs.md
  - ../raw/forensics/file-triage-archives-and-one-liners.md
  - ../raw/forensics/UMDCTF2019-matryoshka-wp.md
updated: 2026-07-28
---

# File-Format and Embedded-Payload Identification

## 适用场景

附件扩展名、magic、容器声明和实际字节布局不一致，或主文件前后/内部嵌有第二载荷；目标是先确认解析边界，再把内嵌对象独立提取。

## 识别信号

- `file`、扩展名和解析器结果冲突。
- EOF 后仍有高熵数据，或容器目录列出未显示对象。
- 某偏移出现新的 magic、压缩流、编码块或重复文件头。

## 最小证据

- 保存原始 SHA-256、文件长度和关键 offset。
- 用格式规范或解析器证明主对象结束位置。
- 提取对象应有独立 magic、结构或解码校验。

## 解法骨架

1. 依次检查 magic、metadata、容器目录、字符串和十六进制边界。
2. 按结构长度而非肉眼猜测确定 section/chunk/EOF。
3. 对每个候选 offset carve 到独立文件，再递归执行格式识别。
4. 建立父对象到子对象的 offset/哈希清单，避免丢失证据链。

## 关键变体

- 尾随/拼接文件：主格式 EOF 后附加第二对象。
- 容器隐藏对象：Office/PDF/归档内部目录或未渲染资源。
- 伪扩展名/多格式 polyglot：不同解析器从不同入口理解同一字节。

## 常见陷阱

- 只改扩展名，没有证明实际格式。
- `strings` 发现线索后未回到 offset 和结构。
- 直接修改原附件，导致证据和偏移不可复现。

## 关联技巧

- [file-signatures-and-flag-artifact-hunting.md](file-signatures-and-flag-artifact-hunting.md)
- [file-triage-archives-and-one-liners.md](file-triage-archives-and-one-liners.md)
- [layered-encoding-and-symbol-mapping-recovery.md](layered-encoding-and-symbol-mapping-recovery.md)
- [structured-document-history-and-hidden-object-recovery.md](structured-document-history-and-hidden-object-recovery.md)

## 原始资料

- [encodings-qr-and-esolangs.md](../raw/crypto/encodings-qr-and-esolangs.md)
- [file-triage-archives-and-one-liners.md](../raw/forensics/file-triage-archives-and-one-liners.md)
- [UMDCTF2019-matryoshka-wp](../raw/forensics/UMDCTF2019-matryoshka-wp.md)
