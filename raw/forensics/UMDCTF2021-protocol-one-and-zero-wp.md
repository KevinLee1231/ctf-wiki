# Protocol One and Zero

## 题目简述

题目给出一份 ICMP 抓包。每个 Echo Request 的数据部分看似只是重复的 `00` 或 `ff`，实际使用两种填充值表示二进制 0 和 1，按包的时间顺序组成隐蔽信道。

## 解题过程

在 Wireshark 中只保留 ICMP Echo Request：

```text
icmp.type == 8
```

逐帧查看 payload。若负载末尾是 `0`（即 `00` 模式）记为比特 `0`，若末尾是 `f`（即 `ff` 模式）记为比特 `1`。也可用 `tshark` 导出：

```bash
tshark -r capture.pcap -Y "icmp.type == 8" \
  -T fields -e data.data
```

严格按抓包顺序连接比特，每 8 位作为一个大端字节转 ASCII，得到：

```text
UMDCTF-{b1n_p1Ng_P0ng}
```

## 方法总结

隐蔽信道分析要关注“通常不影响协议功能、但可被发送端控制”的字段。这里 ICMP 仍能正常完成探测，变化只发生在填充数据。解码时应固定方向和包类型，排除 Echo Reply，否则响应包会复制负载并造成比特重复。
