# Philip 1

## 题目简述

题目给出一份 Linux 内存镜像。需要从进程和 shell 历史中还原 Philip 的远程登录行为，提取留在内存文件系统缓存中的 SSH 私钥，再进入目标环境取得 flag。

## 解题过程

Linux 版 Volatility 2 需要与内核版本匹配的 profile。加载 profile 后先查看进程命令行和 bash 历史：

```bash
vol.py -f memory.lime --profile=<LinuxProfile> linux_psaux
vol.py -f memory.lime --profile=<LinuxProfile> linux_bash
```

输出中能看到 `ssh`、`scp` 及私钥文件的使用痕迹。随后枚举文件缓存并定位密钥 inode：

```bash
vol.py -f memory.lime --profile=<LinuxProfile> linux_enumerate_files | grep -i key
vol.py -f memory.lime --profile=<LinuxProfile> linux_find_file \
  -i <inode-address> -O philip.key
chmod 600 philip.key
```

根据历史记录中的用户名、主机和命令，用该密钥 SSH 登录。远端保存的内容仍经过 Base64：

```bash
ssh -i philip.key <user>@<host>
printf '%s' '<base64-text>' | base64 -d
```

最终得到：

```text
UMDCTF-{G4ll4gh3r_4_1if3}
```

## 方法总结

Linux 内存取证要把“行为证据”和“文件证据”串起来：`linux_psaux`、`linux_bash` 说明用户做过什么，`linux_enumerate_files` 与 `linux_find_file` 才负责恢复实际凭据。提取 SSH 私钥后还要修正权限，否则 OpenSSH 会拒绝使用。
