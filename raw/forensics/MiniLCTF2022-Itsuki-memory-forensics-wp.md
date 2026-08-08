# MiniLCTF2022 含樹イツキの溜息 Writeup

## 题目简述

附件是一份 Windows 虚拟机内存镜像。flag 通过网络传输后被压入加密 ZIP；需要从内存同时恢复运行进程、打开文件、网络会话和登录凭据，并修复损坏的抓包，才能重建完整事件过程。

## 解题过程

先识别系统配置文件并枚举进程、文件对象与连接。关键进程包括 Wireshark、Bandizip、Firefox、命令行和 XChat：

```powershell
vol.py -f memory.raw imageinfo
vol.py -f memory.raw --profile=<profile> pslist
vol.py -f memory.raw --profile=<profile> filescan
vol.py -f memory.raw --profile=<profile> netscan
```

文件扫描显示 Wireshark 打开过一个 PCAP，Bandizip 打开过一个 ZIP；将对应文件对象导出。桌面位图也可从内存恢复，能看到 Wireshark、XChat 对话和 Bandizip 密码输入框，是进程与文件关系的辅助证据。

导出的 PCAP 结构受损，Wireshark 无法完整读取。十六进制检查可见记录间混入重复 `00` 字节；可以手工删除异常字节，也可用 `pcapfix` 重建记录边界。修复后，IRC 明文表明 HS 从 SomeBody 处购买了 flag，并通过另一条 TCP 连接传输压缩包；对话还提示压缩密码是 HS 熟悉的密码。

最后从内存中的登录会话提取凭据。Volatility 2 的社区 `mimikatz` 插件可恢复：

```text
Hs_w4nt5_4_gf
```

使用该字符串解压从内存导出的 ZIP，即可得到 flag。这里的密码并非从流量中直接泄漏，而是由 IRC 语义提示与内存凭据共同确定。

## 方法总结

内存取证的价值在于同时保存“谁在运行、打开了什么、连向哪里、曾处理哪些秘密”。本题应先用进程和句柄建立调查假设，再导出 PCAP/ZIP；损坏文件要在副本上修复，并保留原始镜像。单独看到 IRC、ZIP 或密码都不足以解释全链路，三者关联后才形成可验证结论。
