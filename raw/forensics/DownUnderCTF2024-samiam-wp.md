# SAM I AM

## 题目简述

附件包含从 Windows 注册表导出的 `sam.bak` 与 `system.bak`。SYSTEM hive 中的启动密钥可用于解密 SAM 中的本地账号哈希；恢复 Administrator 的 NTLM 哈希后进行离线口令恢复即可。这是注册表证据恢复，归入 Forensics。

## 解题过程

将两份 hive 交给可解析 SAM 的工具：

```text
lsadump::sam /sam:sam.bak /system:system.bak
```

输出中的 RID `500` 对应内置 Administrator，其 NTLM 哈希为：

```text
476b4dddbbffde29e739b618580adb1e
```

把哈希以一行一个的形式保存后，按 NTLM 模式进行离线字典恢复；官方解法使用模式 `1000`。恢复出的明文密码是：

```text
!checkerboard1
```

按题目格式提交：

```text
DUCTF{!checkerboard1}
```

## 方法总结

Windows 的 SAM 不能只靠 `sam.bak` 独立解出，必须与同机的 SYSTEM hive 配对，因为后者提供解密所需的启动密钥。取证流程应先确认账号 SID/RID，再对准确的哈希做离线恢复；不要把同一机器上其他账号的哈希误归为 Administrator。
