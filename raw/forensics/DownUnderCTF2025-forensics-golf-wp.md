# Forensics Golf

## 题目简述

服务通过 NBD 暴露一块每次连接随机生成的 ext4 文件系统，并对单个客户端发出的字节设置约 24 KiB 上限。根目录有名为 `flag.jpg` 的符号链接，实际 JPEG 位于三层随机目录的叶节点。不能完整下载镜像或递归扫描目录；必须只读取 ext4 元数据中通向该链接和目标文件的块，因此归入 forensics。

## 解题过程

先完成 NBD 协商并仅发送精确的读请求。ext4 的超级块在偏移 1024 处，从中读取 `s_log_block_size`、每组块数和 inode 大小：

```python
block_size = 2 ** (u32le(superblock, 24) + 10)
inode_size = u16le(superblock, 88)
```

读取一个块组描述符，取其 inode table 起始块号；root inode 固定为 2，因此位置为：

```text
inode_offset = inode_table_block * block_size + inode_size * (2 - 1)
```

该镜像的目录和普通文件使用 ext4 extent。inode 的 extent header 位于偏移 40，魔数为 `0xF30A`；从第一条 extent 取得逻辑块号与长度后，只读取对应目录数据块。解析目录项的 `inode`、`rec_len`、`name_len` 和 `name` 字段，在根目录中找到 `flag.jpg`。

`flag.jpg` 的 inode 实际是短符号链接，目标字符串内联在 inode 的数据区。取出该路径并逐段处理：每进入一层随机目录，只读取该目录 inode 与一个目录数据块，直至最后的 JPEG inode。最后读取目标 JPEG 的 extent 数据并打开图像，得到其中的文字：

```text
DUCTF{i_hope_you_learned_a_lot_about_ext4_31890143}
```

官方求解器的读取预算为：超级块、块组描述符、root inode、root 目录、符号链接 inode、三层目录各自的 inode 与数据块，以及最终 JPEG 数据块，共 22,848 字节，低于 24,576 字节限制。最终 JPEG 只承载纯文本 flag，已转写，故不单独保留图片。

## 方法总结

- 核心技巧：从 ext4 元数据建立“root directory → symlink → 目标路径 → file extent”的最短证据链，而不是恢复整块磁盘。
- 识别信号：NBD/磁盘读取有严苛带宽上限，且题面给出或可发现根目录符号链接时，应优先计算 inode、目录项和 extent 所需的最小块集合。
- 复用要点：先从超级块确定块大小和 inode 大小，再按块组描述符定位 inode table；对于短符号链接，目标通常内联在 inode 内，无需再读取数据块。每次 read 前核算字节预算并避免枚举随机目录。
