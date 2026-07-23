# Advantageous Adventures 2

## 题目简述

第二关给出一份无线网络抓包。题面背景暗示有人在附近主动寻找 Wi-Fi，目标是从 802.11 管理帧中恢复被广播的隐藏信息。

## 解题过程

在 Wireshark 中打开抓包，过滤 Probe Request：

```text
wlan.fc.type_subtype == 0x04
```

客户端在扫描无线网络时会发送 Probe Request，其中 SSID 参数位于 tagged parameters。逐帧检查 SSID，或用 `tshark` 批量输出：

```bash
tshark -r capture.pcap -Y "wlan.fc.type_subtype == 0x04" \
  -T fields -e wlan.ssid
```

其中一条探测请求直接携带：

```text
UMDCTF-{sp00ky_sh@rky}
```

## 方法总结

普通 PCAP 取证应先按协议语义筛选，而不是对文件直接跑 `strings`。主动扫描 Wi-Fi 的 Probe Request 可以暴露客户端正在寻找的 SSID，本题正是把 flag 放进这一字段；确认帧类型和 tagged parameter 后即可得到完整证据链。
