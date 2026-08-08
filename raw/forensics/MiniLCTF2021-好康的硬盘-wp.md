# MiniLCTF 2021 - 好康的硬盘

## 题目简述

附件是一条多阶段数字取证链：ZIP 中包含一份带零宽字符的文本和一个 RAR5 压缩包；解开 RAR 后得到磁盘镜像；镜像中的两个已删除文件分别提供视频口令与 SpamMimic 密文。每一阶段的产物都是下一阶段的线索，不能只停留在压缩包爆破。

## 解题过程

首先确认最外层文件头为 ZIP。解压得到 `flag.txt` 和 `luoqian.rar`。`flag.txt` 表面几乎没有内容，但查看 Unicode 码点可见大量 `U+200C`、`U+200D`、`U+FEFF` 等零宽字符。按零宽字符隐写规则解码后得到密码格式提示：

```text
minil****
```

`luoqian.rar` 是 RAR5。用 `rar2john` 提取哈希后，因未知部分明确是四位，直接使用数字掩码而非全字符集爆破：

```bash
rar2john luoqian.rar > rar.hash
hashcat -m 13000 -a 3 rar.hash 'minil?d?d?d?d'
```

恢复密码：

```text
minil4396
```

解压后得到 100 MiB 的 `luoqian.img`。分区表显示 NTFS 分区从扇区 2048 开始，可以用取证工具列出包含已删除项的目录。命令行复现如下：

```bash
mmls luoqian.img
fls -r -o 2048 luoqian.img
```

列表中有两个已删除文件：

```text
27-128-3  奇怪的邮件？.txt
28-128-2  好康的.mp4
```

用 `icat -o 2048 luoqian.img <inode>` 无损导出。视频逐帧检查后，七张短暂出现的画面依次给出数字 `7355608`。文本是一封语句重复、措辞古怪的垃圾邮件，这是 SpamMimic 编码的典型外观：编码器根据隐藏消息选择不同句式和词语，使密文看似普通垃圾邮件。

在 [SpamMimic 的带密码解码页](https://www.spammimic.com/decodepw.shtml) 中粘贴完整邮件，并以 `7355608` 为密码，得到：

```text
MiniLCTF{n3ver_g0nna_L3t_Y0u_dowN}
```

本次归档还直接从原始镜像恢复邮件并重新提交解码，确认了 `Y0u` 的大小写。个别参赛 WP 中出现的 `H5` 是转写错误，不应沿用。

## 方法总结

多阶段取证题应维护清晰的证据链：零宽字符给出掩码，掩码解开 RAR，磁盘镜像恢复已删除文件，视频给出密码，邮件才是最终密文。对磁盘镜像优先使用只读的 `mmls`、`fls`、`icat`，避免挂载写入改变证据；遇到不同 WP 的 flag 冲突时，应回到原始工件重新验证，而不是按多数票猜测。
