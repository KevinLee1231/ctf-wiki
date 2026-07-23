# Advantageous Adventures 3

## 题目简述

第三关仍是一份网络抓包，但 flag 不在常见应用层字段中，而是作为原始数据直接随数据包发送，需要检查未解析的负载。

## 解题过程

先在 Wireshark 中浏览协议层次和长度异常的帧，再使用查找功能搜索 ASCII 文本 `UMDCTF`。也可以让 `tshark` 输出原始数据字段：

```bash
tshark -r capture.pcap -T fields -e data.data
```

对可疑负载做十六进制转 ASCII 后，可以直接读到：

```text
UMDCTF-{wp@_!s_3z}
```

在 Wireshark 的 Packet Bytes 面板中选中对应字节，还可以反向确认这些字符确实属于该帧负载，而不是分析工具生成的注释。

## 方法总结

当题目没有明确上层协议时，应同时检查协议字段和原始负载。搜索只负责定位，最终仍要回到具体数据包确认偏移、所属层和完整边界，避免把抓包元数据或相邻字节误当成答案。
