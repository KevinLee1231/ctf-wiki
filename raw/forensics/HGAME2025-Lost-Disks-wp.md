# Lost Disks

## 题目简述

附件表面上是一个未知文件，实际是一份包含分区表、LVM、LUKS 和 Btrfs 的磁盘镜像。卷组引用了一个缺失物理卷，LUKS 又需要从另一个逻辑卷取得 keyfile，解密后的 Btrfs 还缺少成员盘并设置了非顶层默认 subvolume。目标是在不破坏主数据的前提下逐层恢复存储栈并读取 flag。

## 解题过程

### 1. 识别镜像并建立回环设备

所有会改写 LVM 元数据的操作都应在镜像副本上进行，先保留原始文件并记录哈希：

```bash
sha256sum disk
cp --reflink=auto disk disk.working
file disk.working
```

为工作副本建立带分区扫描的 loop 设备：

```bash
loopdev=$(sudo losetup --find --show --partscan ./disk.working)
echo "$loopdev"
lsblk -f "$loopdev"
```

`file`、分区头和 LVM 扫描会表明分区是 LVM physical volume。进一步查看卷组：

```bash
sudo pvs
sudo vgs
sudo lvs -a -o+devices,layout,cache_mode
```

LVM 报告缺少 UUID 为

```text
NCOmKn-3u2v-pur2-Kyms-XfBI-Zjzf-VVerNx
```

的设备。布局信息显示，缺盘只影响 `nextrainbow/hand1` 的 cache 层，并且模式是 `writethrough`。在该模式下，每次写入同时落到 cache 与 origin LV；[lvmcache 手册](https://man.archlinux.org/man/lvmcache.7)明确区分了这一点：丢失 writethrough cache 不等于丢失 origin 中的数据，而 writeback cache 才可能持有尚未回写的唯一数据。

因此可以在工作副本上解除 cache 关联：

```bash
sudo lvconvert --uncache nextrainbow/hand1
sudo lvchange -a y nextrainbow
```

`--uncache` 会修改 LVM 元数据，这也是前面必须复制镜像的原因；若 `lvs` 显示的不是 writethrough，不能直接套用该结论。

### 2. 把 `hand1` 作为 LUKS keyfile

检查两个逻辑卷：

```bash
sudo file -sL /dev/nextrainbow/hand1
sudo file -sL /dev/nextrainbow/hand2
sudo xxd -l 128 /dev/nextrainbow/hand2
```

`hand2` 开头出现 `LUKS` 魔数，而 `hand1` 内容看似杂乱。结合题目把两者并列放置的结构，可判断整个 `hand1` 逻辑卷就是 keyfile：

```bash
sudo cryptsetup luksOpen \
  /dev/nextrainbow/hand2 recovered \
  --key-file=/dev/nextrainbow/hand1
```

解锁后确认映射内容：

```bash
sudo file -sL /dev/mapper/recovered
```

结果为 Btrfs。

### 3. 降级挂载缺盘的 Btrfs

普通挂载失败后查看内核日志：

```bash
sudo dmesg | tail -n 30
```

题目记录的关键错误为：

```text
BTRFS error: devid 2 uuid a4a78573-1885-46c6-be90-dee5e5b4e2a2 is missing
```

Btrfs 的 `degraded` 选项允许在缺少设备、但现有 chunk 仍满足其 RAID profile 约束时挂载；官方 [Btrfs 管理文档](https://btrfs.readthedocs.io/en/latest/Administration.html)也指出，如果缺失设备上存在唯一的数据 chunk，降级挂载仍会失败。为避免回放日志或进一步修改证据，先只读挂载：

```bash
sudo mkdir -p /mnt/lost-disks
sudo mount -o ro,degraded /dev/mapper/recovered /mnt/lost-disks
```

此时只看到空的 `flag.txt`。检查挂载信息或子卷列表后会发现，文件系统把 `/root` 设置成了默认 subvolume；目标文件实际位于顶层 subvolume。`subvol=` 会覆盖默认选择，因此重新挂载顶层：

```bash
sudo umount /mnt/lost-disks
sudo mount -o ro,degraded,subvol=/ \
  /dev/mapper/recovered /mnt/lost-disks

sudo find /mnt/lost-disks -maxdepth 2 -type f -ls
sudo cat /mnt/lost-disks/flag
```

也可使用等价的顶层 ID：

```bash
sudo mount -o ro,degraded,subvolid=5 \
  /dev/mapper/recovered /mnt/lost-disks
```

目录中的 `flag` 文件即为答案。原 PDF 没有记录 flag 的具体字符串，因此不补造。

完成后按层逆序清理：

```bash
sudo umount /mnt/lost-disks
sudo cryptsetup close recovered
sudo lvchange -a n nextrainbow
sudo losetup -d "$loopdev"
```

## 方法总结

本题要求按真实存储栈逐层定位故障：磁盘镜像 → LVM → 缺失 writethrough cache → LUKS → 缺盘 Btrfs → 默认 subvolume。最关键的判断是“缺盘是否持有唯一数据”，不能看到 cache 就直接删除，也不能在原镜像上试错。使用工作副本、只读降级挂载和显式顶层 subvolume，既能恢复 flag，也保留了可重复取证的证据链。
