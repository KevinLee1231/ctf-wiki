# Packets_dont_lie

## 题目简述

附件包含 TLS 流量和对应的会话密钥日志。目标不是破解 TLS，而是导入密钥后重建 HTTP/2 请求，判断用户最终查询的地点。

## 解题过程

在 Wireshark 的 TLS 协议设置中把 `(Pre)-Master-Secret log filename` 指向附件密钥日志，重新载入抓包后即可解密应用层。过滤 HTTP/2 请求头，可以看到浏览器访问了：

```text
/wiki/Switzerland
/wiki/Kapellbr%C3%BCcke
```

第二条路径是 URL 编码后的 `Kapellbrücke`，即瑞士卢塞恩的卡佩尔廊桥。题目要求提交城市名，因此答案为：

```text
n00bz{lucerne}
```

## 方法总结

拿到 TLS key log 时应优先做合法解密，而不是攻击密码套件。HTTP/2 的有效证据通常位于伪首部 `:method`、`:authority` 和 `:path` 中；路径还需做 URL 解码并结合页面语义解释。
