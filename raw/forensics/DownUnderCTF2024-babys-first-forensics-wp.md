# Baby's First Forensics

## 题目简述

附件是一次 Web 扫描流量的 PCAP，题目只要求识别攻击工具及其版本。决定性证据位于 HTTP 请求的 `User-Agent`，属于网络流量证据恢复，归入 Forensics。

## 解题过程

在 Wireshark 中筛选 HTTP 请求，查看任意一次指向 CGI 路径的请求头。流量包含：

```text
User-Agent: Mozilla/5.0 (Nikto/2.1.6) (Evasions:None) (Test:001022)
```

`Nikto/2.1.6` 同时给出扫描器名称和版本；周围大量针对常见 CGI 路径的探测也与 Web 漏洞扫描行为一致。按题目格式封装即可：

```text
DUCTF{nikto_2.1.6}
```

## 方法总结

流量归因应优先使用协议层的直接标识，例如 `User-Agent`、TLS 指纹、特征请求路径和工具特有报文序列，而不是只凭“请求很多”猜测工具。这里版本号已经出现在原始请求中，不需要爆破、扫描或额外网络查询。
