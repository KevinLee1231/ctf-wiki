# Hackergame2020 室友的加密硬盘 WP

## 题目简述

附件是一段约 2 GB 的磁盘镜像。分区表显示其中包含 `/boot`、Linux swap、LUKS 加密的家目录以及只截取到开头的根分区。题面强调“512 位 AES”，但真正的突破口不是爆破 AES，而是休眠留下的内存内容：swap 中保留了 LUKS 正在使用的主密钥扩展痕迹。

本题需要从磁盘、交换分区和休眠数据中恢复密钥与文件，决定性主障碍是数字证据恢复，因此归入取证方向。

## 解题过程

### 切分镜像并识别分区

`fdisk -l target.img` 给出的关键分区如下：

| 分区 | 起始扇区 | 扇区数 | 含义 |
| --- | ---: | ---: | --- |
| `target.img1` | 2048 | 389120 | ext4 `/boot` |
| `target.img5` | 393216 | 1497088 | Linux swap，含 `SWSUSP1` 休眠镜像 |
| `target.img6` | 1892352 | 1998848 | LUKS 加密家目录 |
| `target.img7` | 3893248 | 12881920 | 根分区，但附件只包含其开头 |

扇区大小为 512 字节，按表中偏移切出需要的分区：

```sh
dd if=target.img of=boot.img bs=512 skip=2048 count=389120 status=progress
dd if=target.img of=swap.img bs=512 skip=393216 count=1497088 status=progress
dd if=target.img of=chome.img bs=512 skip=1892352 count=1998848 status=progress
```

`file` 可以确认 `boot.img` 是 ext4、`swap.img` 是交换/休眠数据、`chome.img` 是 LUKS。`/boot` 中虽然有一个 `keyfile`，内容只是提示，并没有明文密钥。

### 从 swap 恢复主密钥

休眠会把内存页写入 swap。AES 正在工作时，内存中通常不只存在原始密钥，还会存在具有明显轮密钥关系的 expanded key。Radare2 的 `/ca` 可以搜索这类候选：

```text
r2 swap.img
[0x00000000]> /ca
```

题目所说的“512 位 AES”实际对应 AES-XTS 使用的两把 256 位子密钥，总主密钥长度为 512 位。`/ca` 的搜索结果一次显示一段 256 位候选，因此要关注地址相邻的两段并尝试两种连接顺序。成功的顺序是：

```text
e4581675c3f947f7b537a3dd6098e4a5898b0a18c2b3b0f675c61de4106fc6a1
fa01a98089a38f606c148694e7a3509aaccfc165068ed67f5715384b93e56aa6
```

将两行无缝连接并转成 64 字节二进制主密钥：

```sh
printf '%s' 'e4581675c3f947f7b537a3dd6098e4a5898b0a18c2b3b0f675c61de4106fc6a1fa01a98089a38f606c148694e7a3509aaccfc165068ed67f5715384b93e56aa6' \
  | xxd -r -p > key.bin
wc -c key.bin
```

输出必须是 `64 key.bin`。然后利用恢复出的 master key 给 LUKS 增加一个自己知道的口令，再打开和挂载分区：

```sh
sudo cryptsetup luksAddKey chome.img --master-key-file key.bin
sudo cryptsetup luksOpen chome.img chome
sudo mkdir -p /mnt/chome
sudo mount /dev/mapper/chome /mnt/chome
sudo cat /mnt/chome/petergu/flag.txt
```

官方资料还指出一个非预期路径：创建或使用加密卷时，明文口令曾残留在 swap 中，因此对 `swap.img` 执行 `strings` 并尝试可疑字符串也可能打开分区。不过，expanded-key 搜索更符合题目的“休眠内存泄露”机制。

## 方法总结

全盘加密保护的是静态磁盘字节，不会自动清除运行时内存。系统休眠后，密钥、轮密钥甚至口令都可能进入 swap；拿到磁盘镜像的攻击者便能绕过口令强度，直接恢复正在使用的主密钥。取证时应先识别分区与休眠标记，再从 swap 搜索密码学结构，并严格核对候选长度、相邻关系和连接顺序。
