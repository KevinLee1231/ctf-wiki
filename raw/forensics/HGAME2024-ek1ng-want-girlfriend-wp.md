# ek1ng_want_girlfriend

## 题目简述

附件是一份包含 HTTP 流量的抓包文件。flag 不在请求参数或响应头中，而是写在 HTTP 响应传输的一张图片上。决定性步骤是从已取得的 PCAP 证据中导出对象，因此归入 Forensics。

## 解题过程

### 确认 HTTP 传输

用 Wireshark 打开附件，先使用显示过滤器确认存在 HTTP 流量：

```text
http
```

查看相应响应的 `Content-Type`、`Content-Length` 与 TCP 重组结果，可确认服务器返回了一个图像对象。这里不需要手工从分片中拼接二进制内容，Wireshark 可以按 HTTP 会话自动重组对象。

### 导出响应对象

在 Wireshark 中依次选择：

```text
文件 → 导出对象 → HTTP
```

在对象列表中选中图片响应，使用“保存”或“全部保存”导出。如果文件扩展名丢失，可以根据 `Content-Type` 或文件头补上 `.png`/`.jpg`。

打开导出图片，可直接读到：

```text
hgame{ek1ng_want_girlfriend_qq_761042182}
```

官方 PDF 只写了“flag 在图片上”，未展示图片或字符串；上述结果与[HGAME 2024 Week 2 参赛者题解](https://www.cnblogs.com/mumuhhh/p/18012237)中的导出结果一致。关键信息已在正文中给出，不需要依赖外链完成操作。

## 方法总结

- HTTP 文件传输通常跨越多个 TCP 分段；优先使用 Wireshark 的“导出对象”功能，而不是直接复制单个数据包的载荷。
- 对象导出后要用 MIME 类型、文件头和实际解码结果交叉验证，不要仅信任 URL 中的扩展名。
- 本题的图片只承载最终文字；已将其转写为可搜索文本，因此不再保留低信息密度的结果截图。
