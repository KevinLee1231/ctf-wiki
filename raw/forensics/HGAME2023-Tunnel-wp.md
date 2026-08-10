# Tunnel

## 题目简述

题目提供一份包含 IKEv1/IPsec 通信的网络流量，预期方向是分析隧道协议并恢复被传输的数据。不过初版附件存在信息泄露：flag 的明文仍以可打印字符串形式残留在抓包文件中，因此无需先完成 IPsec 解密。

本题的输入是已取得的 PCAP 证据，目标是从证据中恢复事实，所以归入 Forensics。

## 解题过程

先用 `file` 确认附件确实是 pcap/pcapng，再对整个文件做可打印字符串扫描：

```bash
file tunnel.pcapng
strings -a tunnel.pcapng | grep -F 'hgame{'
```

输出中同一段 flag 出现多次：

```text
hgame{ikev1_may_not_safe_aw987rtgh}
hgame{ikev1_may_not_safe_aw987rtgh}
hgame{ikev1_may_not_safe_aw987rtgh}
hgame{ikev1_may_not_safe_aw987rtgh}
```

因此初版题目的 flag 为：

```text
hgame{ikev1_may_not_safe_aw987rtgh}
```

这里应把直接字符串扫描作为取证流程的首轮低成本检查，而不是看到 ESP 就立刻假定所有有效载荷都经过了正确加密。出题方随后发布的 `Tunnel Revenge` 修复了这个非预期，需要从系统捕获中恢复 SA 密钥并真正解密 ESP。

初版明文泄露及其结果可由[出题人补充的 Tunnel 与 Tunnel Revenge 复现](https://crazymanarmy.github.io/2023/01/31/Hgame-2023-week3-Tunnel-%26%26-Tunnel-Revenge-Writeup-CN/)交叉核验；该外部复现记录的关键结论已经完整写入上文，无需依赖链接才能完成本题。

## 方法总结

本题的关键是检查题目实现与预期攻击面之间的落差。处理流量附件时，应先执行 `file`、`capinfos`、`strings`、协议统计等低成本检查，再决定是否进入复杂协议解密。加密协议出现在抓包中，并不等于目标数据一定只以密文存在；附件制作过程中的缓存、日志或重复载荷都可能留下明文。
