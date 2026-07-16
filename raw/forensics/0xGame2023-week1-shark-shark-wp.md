# shark shark

## 题目简述

附件是 FTP 会话的 PCAPNG。FTP 控制连接以明文传输账号、口令和传输命令，文件内容则位于单独的数据连接；需要把控制流中的凭据与数据流中的 ZIP 文件关联起来。

## 解题过程

在 Wireshark 中按协议筛选 `ftp`，或追踪 `tcp.stream eq 0`，可以看到 FTP 登录过程并取得口令：

```text
very_safe_password
```

控制流还会给出 PASV/PORT 与文件传输命令，据此定位数据连接。本题文件数据位于 `tcp.stream eq 4`，流开头四字节为 `50 4b 03 04`，即 ZIP 本地文件头魔数：

![Wireshark 追踪 FTP 数据连接，流首部出现 ZIP 魔数 50 4b 03 04](<0xGame2023-week1-shark-shark-wp/ftp-zip-stream.jpg>)

在“追踪 TCP 流”窗口把显示方式切换为 Raw，并保存服务器到客户端方向的原始字节，即可直接得到 ZIP。若已经复制了纯十六进制文本，可在本地还原并检查文件：

```bash
xxd -r -p stream.hex > transfer.zip
file transfer.zip
unzip -t transfer.zip
```

最后使用控制连接中获得的 `very_safe_password` 解压，读取其中的 flag。

## 方法总结

FTP 的控制连接与数据连接分离，分析时要通过命令和端口信息建立关联。导出文件应优先保存 Raw 流，避免把 Wireshark 的文本展示、方向混合或偏移信息误当作文件数据；文件魔数和解压测试可用于验证重组是否完整。
