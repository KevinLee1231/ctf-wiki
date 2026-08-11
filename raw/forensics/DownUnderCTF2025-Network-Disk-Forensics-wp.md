# Network Disk Forensics

## 题目简述

题目没有提供可下载的磁盘镜像，而是通过 NBD（Network Block Device）导出名为 `root` 的 ext4 块设备。服务端每次连接都会生成多层随机目录和大量诱饵文件，把含 flag 的 JPEG 放进一个随机最深层目录，并在文件系统根目录创建指向它的 `flag.jpg` 符号链接。

## 解题过程

先在隔离的 Linux/虚拟机环境中加载 NBD 客户端，把题目服务的 `root` export 连接为本地块设备，然后只读挂载：

```bash
sudo modprobe nbd
sudo nbd-client -N root <challenge-host> <port> /dev/nbd0
sudo mkdir -p /mnt/ductf-nbd
sudo mount -o ro /dev/nbd0 /mnt/ductf-nbd
```

挂载完成后不必遍历随机目录树。服务端源码明确创建：

```go
symlink := filesystem.Factory.NewSymlink(
    path.Unix.Join(bottomDir.path, flagFileName),
)
challengeDir.Create("flag.jpg", symlink)
```

因此直接复制或打开根目录的符号链接即可：

```bash
readlink /mnt/ductf-nbd/flag.jpg
cp --dereference /mnt/ductf-nbd/flag.jpg ./flag.jpg
```

图像由服务端把 flag 文本绘制到白底 JPEG 中，打开 `flag.jpg` 后读取文字即可。官方解法也给出了 TinyRange 的一次性做法：连接 NBD、挂载，然后导出 `/mnt/flag.jpg`。WP 保留标准 NBD 流程，使读者无需依赖特定封装工具也能复现。

完成后应卸载并断开设备：

```bash
sudo umount /mnt/ductf-nbd
sudo nbd-client -d /dev/nbd0
```

## 方法总结

- 核心技巧：把远程 NBD export 当作本地只读磁盘挂载，并沿根目录符号链接恢复目标图像。
- 识别信号：题面强调“磁盘通过网络提供”，源码使用 NBD 和 ext4；根目录又显式创建 `flag.jpg` symlink。
- 复用要点：先确认 export 名称，再只读挂载；取证时应区分符号链接与真实文件位置，并在结束后卸载、断开 NBD，避免设备残留或误写证据。
