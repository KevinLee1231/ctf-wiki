# Disk Golf

## 题目简述

附件是磁盘镜像压缩包。目标文件仍存在于文件系统中，需要用 Sleuth Kit 列目录、沿 inode 进入用户主目录，再提取文件内容。

## 解题过程

解压 `disk.img.tar.gz` 后先列出根目录：

```bash
fls disk.img
```

输出中的 home 目录 inode 为 `1723`，继续查看：

```bash
fls disk.img 1723
```

在用户 `johnhackerdoe` 的目录树中定位 `flag.txt`，其 inode 为 `262249`。用 `icat` 直接恢复：

```bash
icat disk.img 262249
```

文件内容是八进制编码的 ASCII；按每组八进制数解码后得到：

```text
n00bz{7h3_l0ng_4w41t3d_d15k_f0r3ns1c5}
```

## 方法总结

磁盘取证应通过文件系统元数据定位 inode，再用 `icat` 提取，避免挂载操作改变证据。命令中的 inode 来自本题镜像，复现时应先用 `fls` 验证而非盲目套用。
