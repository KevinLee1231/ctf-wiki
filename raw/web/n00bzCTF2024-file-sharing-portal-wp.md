# File Sharing Portal

## 题目简述

服务用 `tarfile.extractall` 解包用户上传的 TAR，既未限制成员路径，也未拒绝符号链接。可以先路径穿越覆盖系统 cron 脚本获得定时命令执行，再通过符号链接读取任意文件。

## 解题过程

先准备可执行清理脚本，让它把 `/app` 目录写入可读文件：

```bash
#!/bin/bash
ls -la /app > /app/ls.txt
```

构造 TAR 时把该文件的归档名设为：

```text
../../../etc/cron.custom/cleanup-cron
```

并保留执行权限。服务把 TAR 解到 `/app/srv/uploads/<随机目录>/`，路径穿越会覆盖真实 cron 文件。等待定时任务运行后，再制作包含符号链接 `listing.txt -> /app/ls.txt` 的 TAR；上传并通过 `/read/<name>/listing.txt` 读取，获得随机 flag 文件名：

```text
flag_15b726a24e04cc6413cb15b9d91e548948dac073b85c33f82495b10e9efe2c6e.txt
```

最后上传指向该绝对路径的符号链接并读取，得到：

```text
n00bz{n3v3r_7rus71ng_t4r_4g41n!}
```

## 方法总结

归档解包必须同时防范 `../`、绝对路径和链接跳转，并确认每个最终解析路径仍位于目标目录。随机文件名无法抵御已经取得目录枚举和任意文件读取的攻击者。
