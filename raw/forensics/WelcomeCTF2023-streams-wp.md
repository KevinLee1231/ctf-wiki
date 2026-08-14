# Streams

## 题目简述

附件是一组看似普通的 Windows 日志文件。真正的数据不在日志正文，而是藏在 NTFS Alternate Data Streams（ADS）中。十个文件的隐藏流分别保存 Python 程序的一行，按编号拼接并执行后可还原 Flag。

## 解题过程

在 NTFS 文件系统上列出目录中的备用数据流：

```powershell
Get-ChildItem -Force
Get-ChildItem -File | ForEach-Object {
    Get-Item -LiteralPath $_.FullName -Stream *
}
```

也可以在 `cmd` 中使用 `dir /R`。输出会显示类似 `log1.txt:<stream-name>:$DATA` 的条目。逐一读取非默认流：

```powershell
Get-Content -LiteralPath '.\log1.txt' -Stream '<stream-name>'
```

按照日志编号 1 至 10 排列隐藏流内容，将每行拼回一个完整 Python 程序。恢复的程序构造了列表 `lol`，但原脚本没有直接打印最终字符串，因此在末尾补上：

```python
print("".join(lol))
```

运行后输出：

```text
greyhats{str3Am$_HiDD3n_pYth0n_sh3llc0de_b3w@r3}
```

## 方法总结

- 核心技巧：枚举并读取 NTFS ADS，将分散在多个文件隐藏流中的代码按顺序重组。
- 识别信号：Windows/NTFS 语境、普通文件正文像干扰、题面反复强调 streams 或隐藏内容。
- 复用要点：复制到不支持 ADS 的文件系统可能丢失证据；应先在 NTFS 上枚举流并记录“宿主文件、流名、顺序”三项信息。
