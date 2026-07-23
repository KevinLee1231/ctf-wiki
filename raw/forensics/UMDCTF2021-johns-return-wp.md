# John's Return

## 题目简述

题目给出一份 802.11 抓包，并提供无线网络的 SSID 与口令线索。目标是利用 WPA 握手派生会话密钥，解密后续无线数据帧并找到 flag。

## 解题过程

抓包中包含 SSID：

```text
linksys
```

以及完整的 WPA 四次握手。已知口令为：

```text
chocolate
```

在 Wireshark 的协议设置中启用 IEEE 802.11 解密，并添加密钥：

```text
wpa-pwd:chocolate:linksys
```

重新解析后，原先显示为受保护数据的帧会出现上层协议和明文负载。过滤已解密的数据帧并搜索 `UMDCTF`，即可得到：

```text
UMDCTF-{wh3r3_j0hn}
```

命令行也可先用 `airdecap-ng`：

```bash
airdecap-ng -e linksys -p chocolate capture.cap
```

## 方法总结

WPA 解密不仅需要密码，还需要 SSID 和抓包中的握手。三者共同用于派生密钥；缺少握手时，即使知道密码也无法离线解密既有会话。本题应保留原始时间顺序并确认 Wireshark 报告成功解密，再从上层负载中提取答案。
