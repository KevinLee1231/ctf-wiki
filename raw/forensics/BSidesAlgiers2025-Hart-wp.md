# BSidesAlgiers2025 - Hart

## 题目简述

题目给出工业传感器网络抓包 `chall.pcap`。攻击者把 flag 拆成短片段，藏在少量 HART-IP 风格 UDP 报文中，并用 3600 余条普通读取流量制造噪声。决定性工作是找出异常命令码、按消息编号复原顺序，再拼接固定偏移上的字符。

## 解题过程

官方解法给出两个筛选锚点：UDP 源端口为 `5094`，且 UDP payload 的第 12 个字节为 HART `Command 0x23`。`0x23` 本身是合法命令，真正可疑点是其量程上下界被反向设置，因此不能只凭协议名判断恶意流量。

每条命中报文的布局为：

- payload 第 3 个字节：消息编号；
- payload 第 14～16 个字节：最多三个 ASCII 字符；
- 字符槽中的 `0x00` 只是末尾填充。

可以直接按官方命令链提取：

```bash
tshark -r chall.pcap \
  -Y "udp.srcport == 5094 && udp.payload[11] == 0x23" \
  -T fields -e udp.payload | \
python3 -c '
import sys
parts = []
for line in sys.stdin:
    h = line.strip()
    if len(h) < 32:
        continue
    seq = int(h[4:6], 16)
    chunk = bytes.fromhex(h[26:32]).replace(b"\0", b"").decode()
    parts.append((seq, chunk))
print("".join(text for _, text in sorted(parts)))
'
```

本地使用同一字段布局读取抓包，命中 13 个 `0x23` 报文，按编号排序后的结果为：

`shellmates{h4rt_r4ng3_r3v3rs4l_pwn3d}`

该结果同时验证了三个条件：过滤后的包数很小、序号可以形成完整顺序、拼接结果满足比赛 flag 格式。

## 方法总结

- 工控抓包中的合法命令码也可能承载攻击，协议合法性不能替代业务语义检查。
- 面对“海量正常包 + 少量短片段”，先找稳定字段组合，再排序和拼接；直接对全包 `strings` 容易丢失顺序并混入噪声。
- 偏移应同时用“十六进制字符串下标”和“原始字节下标”核对，避免把 `line[26:32]` 误写成 payload 的第 27～32 字节。
