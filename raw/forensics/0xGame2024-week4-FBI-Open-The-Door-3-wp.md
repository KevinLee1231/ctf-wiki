# FBI Open The Door!! 3

## 题目简述

本题沿用 `fish.E01`，要求恢复本地用户 `St5rr` 的明文登录密码。

[原始 E01 检材（百度网盘，提取码 `2vmz`）](https://pan.baidu.com/s/1oo6k9svcSJHaXo-9R5hDJg?pwd=2vmz)

## 解题过程

Windows 本地账户的 NTLM 哈希保存在 SAM 注册表配置单元中，解密所需的系统启动密钥来自 SYSTEM。使用 FTK Imager 从镜像导出：

```text
C:\Windows\System32\config\SAM
C:\Windows\System32\config\SYSTEM
```

以管理员权限运行 Mimikatz，读取离线配置单元：

```text
privilege::debug
lsadump::sam /sam:SAM /system:SYSTEM
```

也可以使用 Impacket 得到同样结果：

```bash
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
```

`St5rr`（RID 1000）对应的 NTLM 哈希为：

```text
ff1dc4eea5e2360bb54c1609138be4ec
```

这是无盐 NTLM，可直接用常见弱口令字典离线恢复。例如：

```bash
hashcat -m 1000 ff1dc4eea5e2360bb54c1609138be4ec /usr/share/wordlists/rockyou.txt
hashcat -m 1000 ff1dc4eea5e2360bb54c1609138be4ec --show
```

得到明文 `zaq!xsw@`，提交：

```text
0xGame{zaq!xsw@}
```

## 方法总结

离线恢复 Windows 本地账户凭据时，SAM 与 SYSTEM 必须成对取得：SAM 保存加密的账户哈希，SYSTEM 提供解密材料。提取出 NTLM 后应优先本地字典攻击并在正文记录原始哈希和恢复命令，不需要依赖在线反查站点。
