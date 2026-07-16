# FBI Open The Door!! 1

## 题目简述

警方取得了嫌疑人 St5rr 的 Windows 计算机磁盘镜像，后续 6 道题共用同一份 `fish.E01` 检材。本题要求提交该镜像的 SHA-256，以确认下载文件未损坏且后续分析基于同一份证据。

[原始 E01 检材（百度网盘，提取码 `2vmz`）](https://pan.baidu.com/s/1oo6k9svcSJHaXo-9R5hDJg?pwd=2vmz)

## 解题过程

在 PowerShell 中对完整的 `fish.E01` 计算 SHA-256：

```powershell
Get-FileHash -LiteralPath "./fish.E01" -Algorithm SHA256
```

也可以使用系统自带的 `certutil`：

```powershell
certutil -hashfile "./fish.E01" SHA256
```

两种方法得到相同摘要：

```text
6d393b09ac01accf27bce07a9c07f5721b9e1e1fd5de1cc8cc1a2581a43e68f5
```

按题目要求包裹后提交：

```text
0xGame{6d393b09ac01accf27bce07a9c07f5721b9e1e1fd5de1cc8cc1a2581a43e68f5}
```

## 方法总结

磁盘取证应先记录原始镜像的密码学摘要，后续挂载、导出文件和工具分析均以只读副本进行。哈希不仅是本题答案，也是证明证据在传输和分析前后未发生变化的基础。
