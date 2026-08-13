# baby autopsy

## 题目简述

题目提供一份 EWF/EnCase 格式的 `plswork.E01` 磁盘镜像，目标是恢复已经删除的文本文件。决定性线索不是普通文件内容，而是 Windows 回收站目录中仍保留的删除文件记录，因此应按磁盘取证流程处理。

## 解题过程

在 Autopsy 中新建案件并把 `plswork.E01` 作为 “Disk Image or VM File” 数据源导入。完成 ingest 后展开第五个卷，在 `$RECYCLE.BIN` 下找到 SID 末尾为 `-1001` 的用户目录。官方解法记录的目标文件名为 `$R0ZR5EW.txt`；直接查看或导出该条目即可恢复内容。

目录定位关系为：

```text
vol5
└── $RECYCLE.BIN
    └── S-1-...-1001
        └── $R0ZR5EW.txt
```

文本内容给出 flag：

```text
grey{autopsy_ftw}
```

## 方法总结

E01 是取证镜像容器，不应当作普通压缩包处理。面对“创建后删除的文件”，应先检查文件系统删除记录和 `$RECYCLE.BIN`；回收站中的 `$R...` 保存原文件数据，配套 `$I...` 条目通常保存原路径与删除时间。本题只需利用 Autopsy 的文件树和内容预览即可完成恢复。
