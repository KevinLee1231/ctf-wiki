# Traffic Engineering

## 题目简述

附件包含一份正常 PCAPNG 和一份首部被破坏的无扩展名文件。第一份抓包通过 ICMP payload 外带 Wi-Fi 密码；第二份实际是 802.11 PCAPNG，只是前四字节被清零。恢复文件头后，需要结合 SSID 与密码生成 WPA/WPA2 预共享密钥，解密无线流量中的 HTTP 请求。

## 解题过程

### 从 ICMP payload 恢复密码

正常 ping 的 payload 大多是固定填充，而异常 Echo Request 的 payload 是“十六进制文本的 ASCII”。只取请求方向、筛选全十六进制 payload 并解码：

```python
from string import hexdigits
from scapy.all import ICMP, Raw, rdpcap

parts = []
for packet in rdpcap("network-capture.pcapng"):
    if not packet.haslayer(ICMP) or packet[ICMP].type != 8:
        continue
    if not packet.haslayer(Raw):
        continue
    raw = bytes(packet[Raw].load)
    try:
        text = raw.decode("ascii")
    except UnicodeDecodeError:
        continue
    if text and all(ch in hexdigits for ch in text) and len(text) % 2 == 0:
        decoded = bytes.fromhex(text)
        if decoded != b"FIEOF":
            parts.append(decoded)

print(b"".join(parts).decode())
```

输出内容为：

```text
ping data exfiltration,
here is the password
you will need it later: 1234567890
```

### 修复损坏的 PCAPNG 首部

无扩展名文件开头为 `00 00 00 00`，其后结构符合 PCAPNG Section Header Block。PCAPNG 正确 magic 是 `0a 0d 0d 0a`。先复制文件，再只覆盖副本前四字节：

```bash
cp corrupted wifi-fixed.pcapng
printf '\x0a\x0d\x0d\x0a' | dd of=wifi-fixed.pcapng bs=1 count=4 conv=notrunc
```

修复后 Wireshark 能识别 IEEE 802.11 帧，并在管理帧中找到 SSID：

```text
Xd
```

### 配置 WPA 解密并查找 HTTP flag

在 Wireshark 的 `Preferences -> Protocols -> IEEE 802.11` 中启用解密，添加 key 类型 `wpa-pwd`，值为：

```text
1234567890:Xd
```

该格式是 `passphrase:ssid`。抓包包含建立会话所需的握手后，Wireshark 会派生 PSK 并解开数据帧。使用过滤器：

```text
http contains "shellmates"
```

在解密后的 HTTP 请求中得到：

```text
shellmates{wp4_h3cker_4nd_traffic_eng1ne3ring}
```

## 方法总结

- 核心技巧：从 ICMP 隐蔽数据中恢复口令，修复 PCAPNG magic，再用 `passphrase:ssid` 解密 802.11 流量。
- 识别信号：ICMP payload 长度异常且只含十六进制字符、未知文件仅首部四字节为零、后续出现 802.11 元数据，分别指向外带、首部破坏和 Wi-Fi 抓包。
- 复用要点：始终修复副本；WPA/WPA2 解密还依赖正确 SSID 和握手帧，只有口令并不足够。
