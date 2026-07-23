# UMDCTF2022 Class Project Writeup

## 题目简述

原题提供一台无法正常启动的 Linux 虚拟机及登录密码，要求定位启动故障并找出 flag。当前公开仓库只保留题面和 flag，没有收录体积较大的 VM 镜像，因此以下过程依据题面、仓库中的官方结果和参赛者对原镜像的文件级复盘整理。

关键事实有两项：`/home/aman_esc/Documents/fork_bomb.bash` 中存在 fork bomb，导致系统启动后迅速耗尽进程资源；同目录的 `admin_notes` 保存了 Base64 编码的 flag。

## 解题过程

不要反复启动故障虚拟机。更稳妥的做法是把虚拟磁盘以只读方式挂载，或用 FTK、Arsenal Image Mounter 一类工具直接检查文件系统。这样既不会触发 fork bomb，也不会改变磁盘证据。

在用户文档目录中可以发现：

```text
/home/aman_esc/Documents/fork_bomb.bash
/home/aman_esc/Documents/admin_notes
```

前者包含典型的 Bash fork bomb：

```bash
:(){ :|:& };:
```

它递归创建进程并通过管道继续派生，解释了虚拟机无法稳定启动的原因。禁用其启动入口或离线移除脚本后，系统即可恢复；不过取 flag 并不要求先修复启动。

读取 `admin_notes` 后，对其中的 Base64 字符串解码：

```bash
printf '%s' '<admin_notes 中的字符串>' | base64 -d
```

得到：

```text
UMDCTF{f0rk_b0mb5_4r3_4_b4d_71m3}
```

[参赛者复盘](https://sutharnisarg.medium.com/umdctf-2022-write-ups-c45cdef017bb)补足了公开仓库缺失的镜像检查过程：作者将 VM 磁盘挂载后直接检查文件系统，并在 `admin_notes` 中发现 Base64 内容。上述关键信息已完整写入正文，无需依赖外链理解解法。

## 方法总结

面对“虚拟机损坏”类取证题，首选离线挂载而不是直接启动。这样既能绕过开机即触发的恶意脚本，也能保留文件时间线。排查时应把“造成故障的文件”和“保存目标信息的文件”分开：fork bomb 解释故障，`admin_notes` 才是 flag 的直接证据。
