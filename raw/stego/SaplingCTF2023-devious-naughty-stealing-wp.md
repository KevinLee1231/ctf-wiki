# Devious Naughty Stealing

## 题目简述

附件是 DNS 流量抓包。攻击者把数据切分后编码进 evil-elpam-encroachers.org 的子域名查询中，形成 DNS 隐蔽信道。决定性任务不是还原普通会话时间线，而是识别并重组被有意藏在查询名中的载荷，因此归为 Stego。

## 解题过程

过滤发往目标域名且不是响应的 DNS 包，只导出完整查询名：

~~~bash
tshark -r logs.pcap \
  -Y "dns.qry.name contains evil-elpam-encroachers.org and dns.flags.response == 0" \
  -T fields -e dns.qry.name
~~~

按数据包时间顺序保留 15 条请求，取每个域名最左侧的 label。字符集中的连字符和下划线表明它是 URL-safe Base64；逐段补齐等号并解码后按原顺序拼接：

~~~python
import base64

out = bytearray()
for label in labels:
    label += "=" * (-len(label) % 4)
    out += base64.urlsafe_b64decode(label)
print(out.decode())
~~~

结果是一封完整 MIME 邮件，其中正文给出：

~~~text
maple{dns_eXfi1trati0n_of_data_15_a_real_pr0blem}
~~~

## 方法总结

分析 DNS 隐蔽信道时应先固定域名、请求方向和时间顺序，再判断编码。不要把域名的其余层级或重传包混进数据。Base64 分段可能各自缺少填充，应逐段补齐；最终还要验证重组结果具有 MIME 头和连贯正文，而不只是碰巧出现 flag 前缀。
