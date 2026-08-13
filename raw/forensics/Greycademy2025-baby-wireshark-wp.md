# Greycademy2025 baby_wireshark

## 题目简述

附件是一份对 Flask 登录接口进行大量口令尝试时留下的 PCAP。题目说明绝大多数凭据都错误，目标是从请求与响应的对应关系中找出唯一成功的一组，并按 `grey{username_password}` 组合 flag。

## 解题过程

先确认流量构成。协议统计显示抓包内共有 1172 个 URL 编码的 HTTP POST 请求，每个请求都提交到 `/login`。直接遍历用户名并不可靠，因为正确性由服务器响应决定，所以应先定位与众不同的响应：

```bash
tshark -r login_capture.pcap \
  -Y "http.response" \
  -T fields -e tcp.stream -e http.content_length -e http.file_data
```

1171 个响应的长度都是 18，正文十六进制解码为 `Wrong credentials!`；仅有一个响应长度为 20，正文是 `Correct credentials!`，其 TCP 流编号为 671。再只显示这一条流：

```bash
tshark -r login_capture.pcap \
  -Y "tcp.stream == 671 && http" \
  -T fields \
  -e frame.number -e http.request.method -e http.request.uri \
  -e http.file_data -e http.response.code -e http.content_length
```

请求正文为：

```text
757365726e616d653d68756e7465727868756e7465722670617373776f72643d616263313233
```

按十六进制解码后得到表单数据：

```text
username=hunterxhunter&password=abc123
```

因此最终 flag 为：

```text
grey{hunterxhunter_abc123}
```

## 方法总结

这题的重点不是从海量请求中盲猜“看起来像真的”凭据，而是用响应差异建立证据。先按状态码、长度或正文对响应分组，再借助 `tcp.stream` 回溯同一会话里的请求，可以把 1172 次尝试迅速缩小到唯一成功样本。对于表单数据，还要区分 Wireshark 展示的十六进制字节与解码后的 URL 编码文本。
