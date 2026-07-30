# L3akCTF 2024 The Curious Case of Raven Morris Writeup

## 题目简述

附件 `raven.dd` 是一份带 DOS 分区表的磁盘镜像。镜像中包含一个可直接读取的 FAT32 分区、一个被异或处理的 Linux 分区，以及分区表未分配空间。最终目标需要串联文件系统元数据、已删除文件和未分配空间中的密钥材料。

## 解题过程

先查看分区布局：

```console
mmls raven.dd
```

关键扇区范围如下：

```text
FAT32:       2048 - 43007
Linux:      43008 - 63487
Unallocated:63488 - 73727
```

对两个分区运行 `fsstat`：

```console
fsstat -o 2048 raven.dd
fsstat -o 43008 raven.dd
```

FAT32 分区结构正常，而 Linux 分区呈高熵，说明它经过了加密或其他可逆变换。

继续枚举 FAT32：

```console
fls -o 2048 raven.dd
fls -o 2048 raven.dd 10
```

`images` 目录中的图片带有 EXIF Comment。以 `judgement.jpg` 为例：

```console
icat -o 2048 raven.dd 51654 | exiftool -Comment -
```

注释提示“钥匙在 empty space”，即应关注未分配空间。`work` 目录还有两个已删除文件：

```text
r/r * 36428: xorkeyfile
r/r * 36430: xor.c
```

恢复并检查它们：

```console
icat -o 2048 raven.dd 36430 > xor.c
icat -o 2048 raven.dd 36428 > xorkeyfile
gcc -o xor xor.c
```

`xor.c` 以 512 字节为一块，用 `xorkeyfile` 对指定扇区区间异或。按源码接口处理 Linux 分区：

```console
./xor raven.dd raven-dec.dd 43008 63487 xorkeyfile
fsstat -o 43008 raven-dec.dd
```

此时 Linux 文件系统已经可以解析。把它单独切出并恢复删除文件：

```console
dd if=raven-dec.dd of=linuxpart bs=512 skip=43008 count=20480
extundelete --restore-all linuxpart
file RECOVERED_FILES/*
```

恢复结果中，一份长文本继续提示把 key 扔到了“space”，另一份是带 `Salted__` 头的 OpenSSL 加密数据。这里的 space 仍指磁盘未分配区域。

检查最后一段未分配空间：

```console
dd if=raven-dec.dd bs=512 skip=63488 | hexdump -C
```

绝大部分字节为零，偏移 `0x18d400` 处出现连续 64 字节非零数据。先切出未分配区，再精确提取：

```console
dd if=raven-dec.dd of=unalloc bs=512 skip=63488
dd if=unalloc of=aeskey bs=1 skip=1627136 count=64
```

这 64 字节不是直接作为二进制 AES key 传入，而是先转成连续十六进制字符串，作为 OpenSSL `-k` 的口令参数。对恢复出的 OpenSSL 密文执行：

```console
openssl enc -d -aes-128-cbc \
  -in aesenc \
  -k "$(od -A n -t x1 -w256 aeskey | tr -d ' ')" \
  -out flag
```

解密后的 `flag` 文件即为题目答案。官方材料没有记录最终 flag 明文，因此这里不凭空补写。

## 方法总结

本题的证据链是：

```text
FAT32 图片 EXIF 提示
→ 恢复 FAT32 已删除的 XOR 程序与 keyfile
→ 解开 Linux 分区
→ 恢复 Linux 删除文件与 OpenSSL 密文
→ 在未分配空间定位 64 字节口令材料
→ AES-128-CBC 解密
```

做磁盘取证时，不应只浏览可见文件。分区表、删除目录项、文件系统孤儿文件和未分配扇区都可能分别保存解题链的一部分；同时要严格区分“二进制密钥”和“把十六进制文本当作口令”这两种 OpenSSL 用法。
