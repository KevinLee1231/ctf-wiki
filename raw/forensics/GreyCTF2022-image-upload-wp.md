# GreyCTF2022 - Image Upload

## 题目简述

题目只给出 `dump.pcap`。网络流量中包含一次上传的 PNG，flag 位于 PNG 的文本元数据块；需要先从 HTTP 会话恢复对象，再检查文件元数据。

## 解题过程

在 Wireshark 使用“导出 HTTP 对象”，或用 tshark 重组上传响应/请求体，恢复出 `ctf.png`。先验证 PNG 签名，避免把截断的 TCP 片段直接交给图像工具：

```bash
file ctf.png
exiftool ctf.png
```

`exiftool` 显示 PNG 的 `tEXt` 块：

```text
Author : akashchandrasekaran_grey{wireshark_exiftool_are_good}
```

因此 flag 为：

```text
grey{wireshark_exiftool_are_good}
```

图像画面本身不参与定位或解码，关键证据已经由可复现的元数据文本完整表达，所以归档不保留一张装饰性 PNG。

## 方法总结

PCAP 中恢复文件时要按 TCP 流重组，不能仅复制单个数据包载荷。得到文件后同时检查格式结构、元数据和嵌入文本；分类依据是从既有流量证据恢复文件与事实，因此属于取证，而不是因载体为 PNG 就机械归到隐写。
