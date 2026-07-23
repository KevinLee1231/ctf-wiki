# UMDCTF2022 Still Cruisin Writeup

## 题目简述

公开仓库只保留了加密的 `production_notes.pdf`，原赛时还提供了受密码保护的虚拟机镜像压缩包。完整攻击链是：破解 PDF 密码、用同一密码解压 VM、离线检查文件系统中的删除文件、根据恢复出的提示执行关键词搜索。

题目放置了大量已删除的音频和图片作为干扰项，但核心不是音频频谱或图片隐写，而是磁盘删除数据恢复与线索关联，因此归入 `forensics`。

## 解题过程

先从 PDF 提取适合 John the Ripper 的哈希并进行口令恢复：

```bash
pdf2john.pl production_notes.pdf > production_notes.hash
john production_notes.hash
john --show production_notes.hash
```

恢复出的同一口令也用于解压 VM 镜像。随后以只读方式挂载虚拟磁盘，或交给支持已删除文件恢复的工具分析。公开复盘指出，FTK 没有展示全部必要删除项，而 R-Studio 找到了更多删除文件。

不要被大量已删除的音频和图片带入逐个隐写、频谱分析的方向。恢复出的某个删除文件包含明确提示，据此对整个文件系统进行关键词搜索，最终命中含 flag 的内容：

```text
UMDCTF{7h3r3'5_4_pl4c3_c4ll3d_k0k0m0}
```

当前仓库缺少 VM 镜像和官方解题脚本；[参赛者复盘](https://sutharnisarg.medium.com/umdctf-2022-write-ups-c45cdef017bb)补足了上述原赛时证据链，包括 PDF 与压缩包共用密码、恢复删除提示、再做关键词搜索。正文已包含理解解法所需的全部要点。

## 方法总结

多层取证题应记录每一层凭据的复用关系：PDF 密码不仅解锁文档，还解锁下一层镜像。面对大量诱饵文件，应先寻找说明性文本、文件名和删除记录中的提示，再决定是否做昂贵的媒体分析。当前公开仓库缺少关键镜像，因此这篇 WP 能准确总结机制与结果，但不能提供可在现有仓库上完整复跑的命令输出。
