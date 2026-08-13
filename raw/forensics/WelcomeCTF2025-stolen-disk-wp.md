# Stolen Disk

## 题目简述

附件包含一个 64 MiB 左右的 Linux 磁盘镜像和一封邮件。镜像主体是正常文件系统，但文件末尾追加了以 `Salted__` 开头的 OpenSSL 密文；邮件给出备份口令 `gr3yc4ts_4r3_my_F4v`。公开赛题提示指定了 AES-256-CBC、PBKDF2 和 100000 次迭代，解密结果是一个 gzip 压缩的 tar 归档。

## 解题过程

先搜索 OpenSSL password-based encryption 的固定前缀：

```bash
rg -aob --fixed-strings 'Salted__' grey_disk_chall.img
```

当前附件中唯一命中位于十进制偏移 `67108864`，即正好 64 MiB。把该位置到文件末尾 carve 出来：

```bash
dd if=grey_disk_chall.img of=backup.enc bs=1 skip=67108864 status=progress
```

使用邮件口令和题目指定参数解密：

```bash
openssl enc -d -aes-256-cbc -salt -pbkdf2 -iter 100000 \
  -in backup.enc -out backup.tar.gz \
  -pass pass:'gr3yc4ts_4r3_my_F4v'
```

解密文件以 `1f 8b 08` 开头，确认是 gzip；继续解包：

```bash
tar -xzf backup.tar.gz
```

归档中的 `flag.txt` 内容为：

```text
grey{trust_N0_On3_but_Gr3YH4Ts}
```

加密参数来自比赛提示，[公开参赛者题解](https://sl-lee.github.io/CTF-Writeups/NUS-Greyhats-Welcome-CTF-2025)也记录了这些参数与操作流程；正文已经保留全部必要值，不依赖读者再次打开外链。

## 方法总结

- 核心技巧：在磁盘镜像正常文件系统末尾搜索 `Salted__`，按精确偏移 carving，再结合邮件口令与 OpenSSL 参数解密嵌套归档。
- 识别信号：镜像工具报告 payload 后仍有额外数据、文件大小越过整齐的分区边界、尾部出现 OpenSSL 固定魔数。
- 复用要点：先记录偏移和魔数，再 carve 原始字节；AES 模式、KDF、迭代数和口令必须全部一致，解密后还要用下一层文件魔数验证结果。
