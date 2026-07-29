# The R3 Pig Problem

## 题目简述

附件只有 `pig.pcapng`。题面称来自 r3kapig 世界的警告被重复发送三次，但直接查看 TCP 流并没有发现完整明文。仓库将它标成 Forensics；真正承载消息的却不是包内容，而是单字节 TCP 包之间的发送间隔，因此按决定性机制归入 Stego。

仓库 checker 的目标为：

```text
r3ctf{th3-thr33-b0dy-pr0blem-h4d-n0-solution}
```

## 解题过程

### 找出异常单字节流

先统计会话、源地址与 TCP 负载长度。下面的显示过滤器能选出唯一明显的人为流量：

```text
tcp && ip.src == 192.168.0.24 && tcp.len == 1
```

共得到 1,506 个包。每个包只承载一个字节，但按负载本身拼接没有意义；观察相邻包时间戳差值后，会出现两个紧密的时间簇：

```text
约 0.02 秒
约 0.10 秒
```

这是一条二元时间隐蔽信道。取中点 `0.06` 秒作为阈值即可稳定区分：

```text
delta < 0.06 -> 0
delta >= 0.06 -> 1
```

1,506 个包产生 1,505 个间隔，也就是 1,505 bit。

### 按 MSB 优先还原字节

可以先用 tshark 导出时间戳：

```bash
tshark -r pig.pcapng \
  -Y 'tcp && ip.src==192.168.0.24 && tcp.len==1' \
  -T fields -e frame.time_epoch > times.txt
```

再把相邻时间差映射为 bit，并按每 8 bit 的高位优先顺序解码：

```python
times = [float(x) for x in open("times.txt") if x.strip()]

bits = []
for previous, current in zip(times, times[1:]):
    delta = current - previous
    bits.append("0" if delta < 0.06 else "1")

bitstream = "".join(bits)
bitstream = bitstream[:len(bitstream) // 8 * 8]

plain = bytes(
    int(bitstream[i:i + 8], 2)
    for i in range(0, len(bitstream), 8)
)
print(plain.decode("ascii", errors="replace"))
```

输出开头为：

```text
My crazy army knows the pigs' secret!
r3ctf{th3-thr33-b0dy-pr0blem-h4d-n0-solution}
Do not answer! Do not answer!! Do not answer!!!
My crazy army knows the pigs' secret! ...
```

抓包结尾落在下一轮重复消息的中间，所以总 bit 数不是 8 的整数倍；截去最后不足一字节的残片即可。第一轮完整消息已经给出 flag：

```text
r3ctf{th3-thr33-b0dy-pr0blem-h4d-n0-solution}
```

## 方法总结

处理 PCAP 时不能只追踪协议字段和 payload。固定长度小包、近似周期发送以及时间差的双峰分布，都是时间隐蔽信道的典型信号。本题先按源地址与 `tcp.len==1` 降噪，再以 $0.06$ 秒阈值量化间隔，按 MSB 优先打包，就能恢复重复明文；包里的单字节内容本身只是掩护。
