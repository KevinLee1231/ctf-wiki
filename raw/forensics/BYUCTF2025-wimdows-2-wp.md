# Wimdows 2

## 题目简述

题目沿用 [Wimdows 虚拟机证据](https://byu.box.com/v/byuctf-wimdows)，要求从攻击者取得 shell 后执行的命令中找出已经带 `byuctf{}` 包装的答案。Windows PowerShell 事件日志保存了被 Base64 编码的命令内容。

## 解题过程

在 `Microsoft-Windows-PowerShell` 相关日志中搜索包含 `-EncodedCommand`、`FromBase64String` 或长 Base64 字段的事件。PowerShell 的 `-EncodedCommand` 默认使用 UTF-16LE，因此应先 Base64 解码，再按 UTF-16LE 解释；若字段来自脚本内部的普通 Base64，则根据字节中的空字节分布判断 UTF-8 或 UTF-16LE。

```powershell
$bytes = [Convert]::FromBase64String($encoded)
[Text.Encoding]::Unicode.GetString($bytes)
```

逐条解码后，其中一段攻击命令直接执行：

```powershell
Write-Output 'byuctf{n0w_th4t5_s0m3_5u5_l00k1ng_p0w3rsh3ll_139123}'
```

因此答案为：

```text
byuctf{n0w_th4t5_s0m3_5u5_l00k1ng_p0w3rsh3ll_139123}
```

## 方法总结

- 核心技巧：从 PowerShell 事件日志定位编码命令，并按 PowerShell 的 UTF-16LE 约定还原脚本内容。
- 识别信号：长 Base64 参数、`EncodedCommand`、异常 PowerShell 子进程或脚本块事件通常意味着命令混淆而非加密。
- 复用要点：先判断编码层和字符编码，不要直接把 Base64 输出当 UTF-8；解码结果还应与进程时间线对应。
