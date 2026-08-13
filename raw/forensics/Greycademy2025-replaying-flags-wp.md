# Replaying Flags

## 题目简述

题目提供浏览器流量 `capture.pcapng` 和 TLS 会话密钥日志 `keylogfile.txt`。目标是在不攻击 TLS 的前提下，利用客户端保存的会话秘密解密 HTTPS 流量，找出参赛者提交过的 flag。

## 解题过程

在 Wireshark 中可进入 `Edit → Preferences → Protocols → TLS`，把 `(Pre)-Master-Secret log filename` 指向 `keylogfile.txt`。同样的过程也可用 `tshark` 复现：

```bash
tshark -r capture.pcapng \
  -o tls.keylog_file:keylogfile.txt \
  -Y 'http2.headers.path == "/api/v1/challenges/attempt" || json' \
  -V
```

解密后可看到 HTTP/2 stream 1 向 `/api/v1/challenges/attempt` 发起 POST。紧随其后的 DATA 帧长度为 80，Wireshark 将其解析为 JSON：

```json
{
  "challenge_id": 36,
  "submission": "grey{me_when_i_type_a_flag_for_ppl_to_retype}"
}
```

本地逐帧复核时，该明文位于第 185 帧；这只是当前抓包中的定位信息，真正稳定的筛选条件是请求路径和 JSON 字段，而不是固定帧号。

## 方法总结

TLS key log 保存的是客户端会话秘密，拿到它后应让分析器正常重组 TCP、解密 TLS、再解析 HTTP/2，没必要从密文中搜索字符串。按应用层语义筛选提交接口可避免被其它浏览流量干扰。最终 flag 为 `grey{me_when_i_type_a_flag_for_ppl_to_retype}`。
