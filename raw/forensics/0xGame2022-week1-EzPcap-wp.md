# week1EzPcap

## 题目简述

附件是一份网络抓包，关键数据通过未加密的 HTTP 传输。flag 被拆入一次请求的查询参数中，因此不需要破解 TLS 或恢复文件。

## 解题过程

在 Wireshark 中使用显示过滤器：

```text
http
```

过滤后只剩少量 HTTP 报文。重点检查对 `check.php` 的 GET 请求；截图可见 URL 中包含 `username` 和 `password` 参数，参数值携带 `0xGame%7B...%7D` 形式的数据。选中该报文后展开 `Hypertext Transfer Protocol`，或右键选择“追踪 HTTP/TCP 流”，将完整查询字符串取出。

最后进行 URL 解码：`%7B` 对应 `{`，`%7D` 对应 `}`。如果 flag 被两个参数分开，应按请求中的先后顺序拼接参数值，而不是把参数名、`&` 或 `=` 一并加入。

## 方法总结

- 核心方法：过滤 HTTP 报文，定位异常 GET 查询参数，并对参数值做 URL 解码和拼接。
- 识别特征：抓包中存在明文 HTTP，URL 里直接出现 `0xGame%7B` 等百分号编码的 flag 前缀。
- 注意事项：应从报文字段或完整流中复制数据，避免因 Wireshark 列表列宽截断而漏字符。
