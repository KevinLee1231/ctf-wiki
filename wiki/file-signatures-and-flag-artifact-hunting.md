---
type: family
tags: [forensics, file-triage, artifact, family]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/file-signatures-and-flag-artifact-hunting.md
updated: 2026-07-11
---

# File Signatures and Flag Artifact Hunting

## 作用边界

本页是文件首检 family。首轮不知道文件真实类型，或者怀疑 flag 藏在 metadata、文件尾部、未引用对象、删除工件、日志碎片、常见编码层中时，用它先建立“文件到底是什么、还有什么没看”的分流清单。

边界在于“文件内部 artifact 取证”：如果题目只是签到、一行命令、hash/cipher 识别、API 枚举或轻量压缩包分流，先转 [file-triage-archives-and-one-liners.md](file-triage-archives-and-one-liners.md)；如果已经确认是损坏归档、ZipCrypto、磁盘/容器恢复，转 [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md)。

## 识别信号

- 文件扩展名和 magic bytes 不一致，或 `file` 输出过于笼统。
- `strings` 出现 flag-like 片段、base64/hex、路径、用户名、时间戳、PDF object、EXIF 字段。
- binwalk/7z 能列出嵌入内容或 trailing data。
- 题目看似没有方向，只给一个奇怪文件、压缩包、图片、PDF 或日志。

## 最小证据

- 至少记录 `file`、前 32 字节 hex、`strings` 前后文、metadata 结果。
- 如果发现编码串，要能正向解码并说明是否还有下一层。
- 如果发现嵌入/尾随数据，要导出成独立文件再重新 `file`。

## 分流流程

1. 首检：`file target; xxd -l 64 target; strings -a target | head; exiftool target`。
2. 容器检查：`7z l`、`binwalk`、`mutool info`、`ffprobe`、`identify -verbose`；`pdfinfo` 仅在已安装时使用。
3. 搜索 flag 位置：metadata、comments、unallocated/trailing bytes、PDF hidden object、browser/log fragments。
4. 编码剥离：base64、hex、ROT13/ROT18、URL、gzip/zlib、xor single-byte。
5. 每剥一层都保存输出并重新识别类型。

## 首检路线分流

| 首检信号 | 下一跳判断 |
|---|---|
| Magic mismatch | 扩展名不可信，以 header/footer 和结构为准。 |
| Metadata flag | PDF/Image/Office/Audio 的 Author、Comment、Keywords 常见。 |
| Trailing data | JPEG/PDF/ZIP 后附额外字节，需按 EOF marker 切出。 |
| Common encoding | base64/hex/ROT18 常是最终或倒数第二层。 |

## 常见陷阱

- 只运行 `strings | grep flag`，没有看 metadata、尾部和嵌套文件。
- 解码一次后不重新 `file`，错过多层容器。
- 用自动工具覆盖原件，导致证据链不可复现。
- 忽略大小写、leet、反转、ROT18 等轻量变形。

## 关联技巧

- [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md)
- [file-triage-archives-and-one-liners.md](file-triage-archives-and-one-liners.md)
- [3d-printing.md](3d-printing.md)
- [audio-frequency-and-archive-stego.md](audio-frequency-and-archive-stego.md)
- [blockchain-and-transaction-forensics.md](blockchain-and-transaction-forensics.md)
- [cross-domain-forensics-technique-map.md](cross-domain-forensics-technique-map.md)
- [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md)

## 原始资料

- [file-signatures-and-flag-artifact-hunting.md](../raw/forensics/file-signatures-and-flag-artifact-hunting.md)
