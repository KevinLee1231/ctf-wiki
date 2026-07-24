# UMDCTF 2018 - Eternally Crying

## 题目简述

附件是 Windows Sysmon Operational EVTX。题面称主机感染了勒索软件，要求从事件日志中恢复恶意程序泄露的消息。仓库开发代码还给出了勒索文件后缀 `.barkbark` 和通过注册表写入信息的行为。

## 解题过程

先按事件类型统计日志。与题目时间线对应的是 Sysmon Event ID 13，即注册表值被设置的事件。相关记录具有同一可疑进程：

```text
Image: C:\Windows\bacwrkzT.exe
```

筛选该进程后，再按 `EventRecordID` 排序。连续的有效记录范围为 `17051` 至 `17130`，每条记录的注册表目标路径末端携带一个字符。按事件顺序取出这些字符并连接，就能从大量重复注册表噪声中恢复消息。

可用的处理逻辑如下：

```python
events = [
    event for event in parsed_events
    if event["EventID"] == 13
    and event["Image"].lower().endswith(r"\bacwrkzt.exe")
]
events.sort(key=lambda event: int(event["EventRecordID"]))
message = "".join(event["TargetObject"].rsplit("\\", 1)[-1] for event in events)
```

恢复出的 flag 为：

```text
UMDCTF-{its-quite-noisy-here}
```

其 SHA-256 与 `README.md` 中的摘要一致。

## 方法总结

EVTX 题的重点不是对所有字段做盲目字符串搜索，而是先用事件 ID、进程路径和连续记录号缩小时间线，再寻找攻击者如何把字符编码到字段中。开发代码用于确认行为模型，最终字符顺序仍应以日志记录顺序为准。
