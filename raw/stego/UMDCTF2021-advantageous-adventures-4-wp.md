# Advantageous Adventures 4

## 题目简述

第四关给出一份网络抓包。数据包负载内容本身没有明文，但发送脚本通过“每个数据包的长度”编码字符，并重复发送同一长度以便观察。

## 解题过程

仓库中的生成逻辑对 flag 的第 $i$ 个字符 $c_i$ 构造：

```python
payload = str(i) * ord(c_i)
```

也就是说，负载由索引数字重复组成，重复次数就是字符的 ASCII 值。每个字符对应的数据包又发送 100 次，因此应先按索引或连续相同长度去重。

解码时按时间把 100 个相同负载视为一组。第 $i$ 组中，每个索引文本 `str(i)` 被重复了 `ord(c_i)` 次，所以字符值应为 `len(data) // len(str(i))`：

```python
from scapy.all import rdpcap, Raw

groups = []
for packet in rdpcap("capture.pcap"):
    if Raw not in packet:
        continue
    data = bytes(packet[Raw].load)
    if groups and groups[-1] == data:
        continue
    groups.append(data)

plain = "".join(
    chr(len(data) // len(str(index)))
    for index, data in enumerate(groups)
)
print(plain)
```

索引超过一位时不能把 payload 总长度直接当 ASCII 值，也不能只取首字节；必须除以十进制索引文本自身的长度。恢复结果为：

```text
UMDCTF-{crypt0_1n_wp@?}
```

## 方法总结

网络隐写不一定修改字段值，也可能利用长度、时序或重复次数。本题真正的信息量在 payload 长度，内容只是标识字符位置。读取前要先理解生成脚本并去掉 100 次冗余，否则会得到大量重复字符。
