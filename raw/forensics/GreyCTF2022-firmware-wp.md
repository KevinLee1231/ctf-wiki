# GreyCTF2022 - Firmware

## 题目简述

附件 `firmware.img.gz` 是压缩的磁盘镜像。flag 不依赖固件代码执行，而是藏在根文件系统配置中；决定性步骤是识别分区和 SquashFS，并恢复文件系统内容，因此归类为取证。

## 解题过程

先只读解压镜像并查看分区表：

```bash
gzip -dc firmware.img.gz > firmware.img
fdisk -l firmware.img
```

第二分区起始于 36864 扇区。按默认 512 字节扇区计算偏移 $36864\times512$，提取该分区后识别为 xz 压缩的 SquashFS：

```bash
dd if=firmware.img of=rootfs.img bs=512 skip=36864 status=none
file rootfs.img
unsquashfs -cat rootfs.img etc/inittab
```

无需挂载或修改镜像，`unsquashfs -cat` 即可直接读取目标文件。`/etc/inittab` 的注释中写着：

```text
# you found me! grey{inittab_1s_4n_1mp0rt4nt_pl4c3_t0_l00k_4t_wh3n_r3v3rs1ng_f1rmw4r3}
```

## 方法总结

固件镜像先做容器识别：压缩层、分区表、文件系统类型逐层剥离。偏移必须由扇区号与扇区大小计算，尽量使用只读提取工具；嵌入式启动问题可优先检查 `/etc/inittab`、init 脚本、启动服务和环境配置。
