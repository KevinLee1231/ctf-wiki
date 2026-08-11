# DownUnderCTF 2023 Rogueful Helper Writeup

## 题目简述

题目提供一份约 1.6 GB 的加密 Windows 调查包，要求找出在 `2023-08-26 15:32:20` 完成的侦察任务所使用的 ICMP payload。官方归档记录的压缩包 SHA-256 为 `e7e12a1e421d685f8f1ecfebc33faa7bff5696b6b7c18559dd2389b6e5d1ac60`，解压密码为 `quooz6cuin5Aiw2aiRue9een2eimuviem2ibi2Ahr7Chiepeof9oxuz5oofeu1oo`。

## 解题过程

Windows 事件日志和 MFT 时间线能够看到 Npcap、Kaseya VSA 路径等痕迹，说明网络侦察工具是 Nmap；但事件日志没有记录完整命令，Nmap XML 结果也已不在主机上。继续检查 Kaseya 相关应用数据，可以找到 SQLite 数据库 `audit.s3db`。

用 SQLite 工具打开数据库，先列出表和字段，再按完成时间筛选任务。目标记录是 task 3；它的 `args` 字段保留了完整 Nmap 参数。无需猜测历史命令，只需从该字段提取 ICMP payload：

```text
cHd5cmVxAWFhYWFhYWFhYWFhYWFhYWFhYWFhYWFhYWE=
```

末尾 Base64 填充符在题目判定中可有可无。解码可辅助确认它确实是 32 字节探测载荷：

```powershell
$payload = 'cHd5cmVxAWFhYWFhYWFhYWFhYWFhYWFhYWFhYWFhYWE='
$bytes = [Convert]::FromBase64String($payload)
[BitConverter]::ToString($bytes)
```

结果以十六进制 `70-77-79-72-65-71-01` 开头，后跟连续的 `61` 字节。按题目格式提交：

```text
DUCTF{cHd5cmVxAWFhYWFhYWFhYWFhYWFhYWFhYWFhYWFhYWE=}
```

## 方法总结

本题的关键是从“操作系统没有进程命令行日志”转向应用自身审计数据库。工具输出文件被删除不代表任务参数消失；Kaseya 的 `audit.s3db` 仍保留任务、时间和完整参数。调查时应先用时间线定位软件，再沿软件的数据目录寻找结构化审计记录。
