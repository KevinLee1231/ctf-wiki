# Do not enter

## 题目简述

附件是经过 gzip 压缩的磁盘镜像。镜像包含多个 Linux 分区，其中一个 ext4 分区的卷标为 `Do_not_enter`；卷标既是题目提示，也是定位目标分区的决定性证据。

## 解题过程

先解压镜像，再通过 loop 设备的 `-P` 选项自动扫描分区表。不要直接挂载整个镜像，应先用 `lsblk -f` 查看各分区的文件系统和卷标：

```console
$ gzip -dk do_not_enter.dd.gz
$ loopdev=$(sudo losetup --find --show --partscan do_not_enter.dd)
$ lsblk -f "$loopdev"
NAME      FSTYPE FSVER LABEL        UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
loop0
├─loop0p1 ext4   1.0   UserShare    5a6be8f0-43f9-4020-a729-510d6d57e95b
├─loop0p2 ext4   1.0   Do_not_enter 643298ec-2a07-4681-9555-addf90de8ae1
├─loop0p3
├─loop0p5 ext4   1.0   WebServer    f965eed6-3de2-4533-8e06-2c816f9e4574
└─loop0p6 ext4   1.0   SysLogs      650ce632-c57e-41c6-8a3b-c6bf3d4e2193
$ sudo mkdir -p /mnt/do-not-enter
$ sudo mount "${loopdev}p2" /mnt/do-not-enter
$ sudo grep -R "0xGame" /mnt/do-not-enter
/mnt/do-not-enter/syslog:0xGame{WoW_y0u_fouNd_1t?_114514}
$ sudo umount /mnt/do-not-enter
$ sudo losetup -d "$loopdev"
$ sudo rmdir /mnt/do-not-enter
```

挂载卷标对应的第二分区后，对文件内容递归搜索 flag 前缀，在 `syslog` 中得到结果。完成后依次卸载文件系统并释放 loop 设备，避免镜像仍被占用。

## 方法总结

- 核心技巧：解析磁盘镜像分区表，依据文件系统卷标选择目标分区并挂载检索。
- 识别信号：原始 `.dd` 镜像、多个分区和具有题意的卷标。
- 复用要点：先 `lsblk -f` 或使用分区分析工具建立分区映射，再挂载具体分区；取证结束后要卸载并释放 loop 设备。
