# 简单的vmdk取证

## 题目简述

附件是一份 Windows XP 的 VMDK 虚拟磁盘。目标是从磁盘中的本地账户信息恢复 Administrator 密码。关键证据位于 NTFS 分区的 Windows 注册表配置单元，而不是普通文档文件。

## 解题过程

### 提取本地账户哈希

用 Magnet AXIOM 等取证工具载入 VMDK，在 Windows 用户账户证据中定位 `Administrator`。对应证据源来自：

```text
Windows XP Professional.vmdk
└─ Partition 1 (Microsoft NTFS)
   └─ WINDOWS/system32/config/SAM
```

工具解析出的 Administrator NTLM 哈希为：

```text
DAC3A2930FC196001F3AEAB959748448
```

也可以不依赖取证软件的账户解析功能：从镜像中导出 `SAM` 与 `SYSTEM` 注册表配置单元，再运行 Impacket 的 `secretsdump.py`，利用 `SYSTEM` 中的启动密钥解密 `SAM` 中保存的本地账户哈希。

```powershell
secretsdump.py -sam SAM -system SYSTEM LOCAL
```

### 恢复密码并校验

将 NTLM 哈希提交到离线字典或哈希库，可得到明文：

```text
Admin1234
```

NTLM 实际是密码 UTF-16LE 编码后的 MD4。可在本地重新计算，确认该明文与镜像中的哈希严格对应：

```python
from Crypto.Hash import MD4

password = "Admin1234"
ntlm = MD4.new(password.encode("utf-16le")).hexdigest().upper()
assert ntlm == "DAC3A2930FC196001F3AEAB959748448"
print(password)
```

官方 PDF 还指出，可以用 7-Zip 等工具直接浏览 VMDK 并导出相关文件。PDF 没有展示带 `hgame{...}` 外壳的结果，因此这里忠实记录可验证的题目答案 `Admin1234`，不自行猜测 flag 包装格式。

## 方法总结

- VMDK 是磁盘证据，应优先只读加载或用取证工具解析，避免启动原虚拟机改变文件系统状态。
- Windows 本地密码恢复通常需要成对取得 `SAM` 和 `SYSTEM`；只有 `SAM` 时未必能解出账户哈希。
- 在线哈希库只负责查找候选明文，最终仍应本地计算 NTLM 并与原哈希逐字节核对。
- 题解未给出 flag 外壳时，应记录确切恢复出的密码，不能擅自写成看似合理的 `hgame{...}`。
