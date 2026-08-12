# DownUnderCTF 2021 - The File is Lava

## 题目简述

题目提供一台疑似参与内部文件外传的 Windows 工作站镜像。需要从用户最近访问文件的痕迹中找到异常，即使目标文档已经不在磁盘上也要恢复其历史路径信息。关键证据是 Windows Shell Link（`.lnk`）文件：打开本地文件时，系统会在用户的 Recent 目录留下包含目标路径、卷信息和时间戳的快捷方式工件。

## 解题过程

应在镜像副本或只读挂载上分析，避免直接启动原始虚拟机改变时间戳。目标用户的 Recent 目录为：

```text
C:\Users\ductf\AppData\Roaming\Microsoft\Windows\Recent
```

使用 Eric Zimmerman 的 `LECmd` 批量解析 `.lnk` 并输出 CSV：

```powershell
LECmd.exe -d "C:\Users\ductf\AppData\Roaming\Microsoft\Windows\Recent" `
  --csv "E:\LnkOutput" -q
```

在 CSV 中按目标文件名搜索 `Executive Management report`。该行除了访问时间，还保留了两个异常的 Base64 片段，分别出现在卷标和本地目录相关字段：

```text
RFVDVEZ7eTB1X2YwdW5kX3Ro
M19tMXNzMW5nX2wxbmt9
```

按字段顺序连接后解码：

```bash
printf '%s' 'RFVDVEZ7eTB1X2YwdW5kX3RoM19tMXNzMW5nX2wxbmt9' | base64 -d
```

得到：

```text
DUCTF{y0u_f0und_th3_m1ss1ng_l1nk}
```

这里不需要依赖已经失踪的原文档；`.lnk` 自身保存的 LinkInfo 和字符串字段就是证据来源。

## 方法总结

Windows Recent `.lnk` 是文件访问与移动调查中的高价值工件，目标文件删除后仍可能保留驱动器卷标、网络共享、本地路径、文件大小和多组时间戳。遇到“文件被复制或外传”的镜像题时，应把 LNK、Jump List、ShellBag 和 RecentDocs 作为相互验证的入口，并始终在只读副本上解析，避免启动镜像污染证据。
