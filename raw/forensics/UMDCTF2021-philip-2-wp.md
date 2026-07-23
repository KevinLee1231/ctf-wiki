# Philip 2

## 题目简述

第二份 Linux 内存镜像中，关键进程是 Thunderbird。需要恢复其邮件配置和附件，从邮件数据库里找到 ZIP 口令，再从缓存文件中重组附件并打开其中的 PDF。

## 解题过程

先用进程和网络插件定位 Thunderbird：

```bash
vol.py -f memory.lime --profile=<LinuxProfile> linux_psaux
vol.py -f memory.lime --profile=<LinuxProfile> linux_netstat
```

进程 PID 为 `2084`。用 `linux_lsof -p 2084` 查看打开文件，可定位 Thunderbird profile。恢复 profile 中的 `global-messages-db.sqlite` 后搜索邮件正文，能找到附件名 `Invoice.zip` 及密码：

```text
bigchungus4life
```

仅恢复数据库不够，附件主体通常存放在邮箱数据文件或缓存中。枚举并导出 profile 下的相关 inode，使用邮件解析器读取 MIME 消息，从 Base64 编码的附件段还原 `Invoice.zip`：

```bash
vol.py -f memory.lime --profile=<LinuxProfile> linux_enumerate_files \
  | grep -i thunderbird
vol.py -f memory.lime --profile=<LinuxProfile> linux_find_file \
  -i <inode-address> -O recovered-mailbox
```

用已知密码解压并打开 PDF，得到：

```text
UMDCTF-{M3g4_Ch4#g4$}
```

## 方法总结

邮件取证不能只看索引数据库。数据库适合搜索主题、正文、附件名和口令，真正附件还可能位于 mbox、IMAP 缓存或 MIME 数据中。本题应先通过 PID 与打开文件确定 profile，再恢复数据库和消息体，最后按 MIME 编码重组附件。
