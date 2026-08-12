# 加密的 U 盘

## 题目简述

附件包含同一块 LUKS 加密 U 盘在两天的镜像。第一天的口令已知，第二天只修改了口令，文件系统中加入了 flag。LUKS 不是直接用口令加密数据：口令解锁 keyslot 中的卷主密钥，真正的数据加密始终使用主密钥。若设备只是执行 `luksChangeKey`，主密钥不会改变，因此旧镜像可用于恢复它。

## 解题过程

### 从旧镜像提取主密钥

先把第一天磁盘镜像映射为 loop 设备，确认 LUKS 分区后，用已知旧口令解锁并导出 master key：

```bash
sudo losetup --find --partscan --show day1.img
# 假设返回 /dev/loop0，LUKS 位于第一个分区
sudo cryptsetup luksDump --dump-master-key /dev/loop0p1
```

`luksDump --dump-master-key` 的输出是十六进制。将其无空格地保存并转换为原始字节，例如：

```bash
tr -d ' \n' < master-key.hex | xxd -r -p > master-key.bin
```

题目卷使用 512 位主密钥，因此输出文件应为 64 字节；长度不符说明复制时混入了标题或换行。

### 用主密钥打开新镜像

映射第二天镜像，然后绕过未知的新口令，直接提供卷主密钥：

```bash
sudo losetup --find --partscan --show day2.img
# 假设返回 /dev/loop1
sudo cryptsetup luksOpen \
  --master-key-file master-key.bin \
  /dev/loop1p1 day2

sudo mkdir -p /mnt/hg2021-day2
sudo mount -o ro /dev/mapper/day2 /mnt/hg2021-day2
```

在只读挂载的文件系统中找到：

```text
flag{changing_Pa55w0rD_d0esNot_ChangE_Luk5_ma5ter_key}
```

完成取证后按相反顺序卸载并关闭映射：

```bash
sudo umount /mnt/hg2021-day2
sudo cryptsetup luksClose day2
sudo losetup -d /dev/loop0 /dev/loop1
```

## 方法总结

- 核心技巧：从可解锁的旧 LUKS 头部恢复卷主密钥，再用它直接解锁只换过口令的新镜像。
- 识别信号：同一卷的前后镜像、已知旧口令、只声明“修改密码”而未重新加密数据。
- 复用要点：口令与数据密钥是两层机制；改 keyslot 不等于轮换主密钥。取证时应只读挂载，并核对导出密钥的字节长度。
