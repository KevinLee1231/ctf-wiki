# DownUnderCTF 2022 DFIR Investigation 2 Writeup

## 题目简述

本题沿用 `DFIR Investigation 1` 的 Windows 取证镜像，要求回答用户 `Challenger` 第一次打开 `passwd.txt` 的时间，以及文件中保存的字符串。flag 格式为 `DUCTF{hh:mm:ss_string}`。

镜像没有直接收录 `passwd.txt`，因此需要关联两类 NTFS 证据：Recent 目录中的快捷方式负责确定打开时间，`$MFT` 记录中的 resident `$DATA` 负责恢复文件内容。

## 解题过程

### 用 Recent LNK 确定首次打开时间

Windows Explorer 打开文档后，通常会在用户的 Recent 目录生成对应的 `.lnk`。本题目标位于：

```text
C:\Users\Challenger\AppData\Roaming\Microsoft\Windows\Recent\passwd.lnk
```

从镜像中提取该文件并用 LNK 解析器（如 LECmd）查看：

```powershell
.\LECmd.exe -f .\passwd.lnk
```

需要区分两组时间：快捷方式自身的创建时间反映 Recent 项何时生成，也就是用户首次打开目标文件的时间；目标文件时间则描述 `passwd.txt` 自身。解析结果给出的首次打开时间为：

```text
08:25:27
```

### 从 `$MFT` 的 resident data 恢复内容

`passwd.txt` 很小。NTFS 对足够小的文件可以不分配独立数据簇，而是把内容直接放进该文件 MFT 记录的 `$DATA` 属性，这称为 resident data。文件在普通目录视图中缺失，并不意味着其内容没有留在 `$MFT`。

从镜像根目录提取 `C:\$MFT`，用支持显示 resident data 的 MFT 解析器定位：

```text
C:\Users\Challenger\Desktop\passwd.txt
```

其 resident `$DATA` 内容为：

```text
R3sident!al
```

组合时间与字符串即可得到：

```text
DUCTF{08:25:27_R3sident!al}
```

## 方法总结

本题的核心是跨工件关联。Recent LNK 能证明文件何时被用户打开，而 `$MFT` 的 resident `$DATA` 能在原文件未被采集时保留小文件内容。取证时应明确每个时间戳和数据字段所描述的对象，避免把目标文件创建时间误当作首次访问时间。
