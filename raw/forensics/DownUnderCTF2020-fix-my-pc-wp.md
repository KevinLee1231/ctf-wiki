# DownUnderCTF 2020 - fix-my-pc

## 题目简述

题目给出一份损坏电脑的 `qcow2` 磁盘镜像和对应的内存转储。磁盘包含一个未加密的启动分区，以及两个使用 LUKS 加密的分区；直接挂载只能看到加密容器，真正的突破口是从内存中恢复仍然驻留的 AES 密钥，再沿系统配置、用户操作痕迹和 Git 历史还原完整证据链。

## 解题过程

### 识别磁盘结构

先用 NBD 把 `qcow2` 暴露成块设备，再查看分区表：

```bash
sudo modprobe nbd max_part=8
sudo qemu-nbd --connect=/dev/nbd0 system.img
sudo fdisk -l /dev/nbd0
```

检查第二、第三分区会发现它们都是 LUKS 容器。`cryptsetup luksDump` 给出的关键参数是 LUKS1、`aes-xts-plain64`、SHA-256，并且主密钥长度为 512 bit：

```bash
sudo cryptsetup luksDump /dev/nbd0p2
```

这说明普通口令爆破不是合理主线；题目同时提供内存转储，暗示应寻找使用过程中留在内存里的密钥材料。

### 从内存恢复 LUKS 主密钥

对内存镜像运行 `findaes`，可以定位到两份相邻的 AES-256 key schedule。LUKS 的 XTS 模式使用两把 AES 密钥，所以把两段各 32 字节的结果按工具输出顺序拼接，即得到 64 字节主密钥文件 `master.key`。

```bash
findaes memory.raw
# 将找到的两段 32 字节 key schedule 还原为原始字节并依次写入 master.key
sudo cryptsetup --master-key-file master.key \
    luksOpen /dev/nbd0p2 cryptroot
sudo mkdir -p /mnt/root
sudo mount /dev/mapper/cryptroot /mnt/root
```

这里必须使用 `--master-key-file`，因为恢复的是 LUKS 数据加密主密钥，不是用户输入的 passphrase。

### 解锁 home 分区

根文件系统中的 `/etc/fstab` 表明第三分区挂载为 `/home`；`/etc/crypttab` 则给出了该分区所用 key file 的路径。于是直接使用镜像中的密钥文件打开第三分区：

```bash
sudo cryptsetup --key-file /mnt/root/etc/crypttab.d/home.key \
    luksOpen /dev/nbd0p3 crypthome
sudo mkdir -p /mnt/home
sudo mount /dev/mapper/crypthome /mnt/home
```

### 串联用户痕迹与 Git 历史

`/mnt/home/bob` 中存在 SSH 私钥，`.ash_history` 又记录了用户曾克隆下面的仓库：

```text
git@github.com:cornochips/configs.git
```

把私钥权限设为仅本人可读后，用它克隆仓库，再检查所有提交而不是只看当前工作树：

```bash
chmod 600 bob_id_rsa
GIT_SSH_COMMAND="ssh -i $PWD/bob_id_rsa -o IdentitiesOnly=yes" \
    git clone git@github.com:cornochips/configs.git
cd configs
git log --all --oneline
git log -p --all
```

历史提交中保留了后来被删除的配置内容，最终恢复出：

```text
DUCTF{aT_l3ast_I_had_A_B3ck8p_y4n63xOVX4A}
```

## 方法总结

- 核心技巧：从内存中的 AES key schedule 恢复 LUKS-XTS 主密钥，随后利用系统配置解锁级联加密分区。
- 识别信号：磁盘镜像与同机内存转储同时出现、分区显示为 LUKS 时，应优先考虑运行态密钥恢复，而不是盲目爆破口令。
- 复用要点：恢复的主密钥要通过 `--master-key-file` 使用；挂载后还应继续检查 `fstab`、`crypttab`、shell history、SSH 密钥和 Git 全历史，不能把“成功解密磁盘”误当成证据链终点。
