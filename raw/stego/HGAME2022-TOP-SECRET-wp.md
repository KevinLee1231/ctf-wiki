# TOP SECRET

## 题目简述

原题是一条连续的三阶段隐写链：先从 PCAP 中按 DNS 查询名的大小写恢复有序字符，再把 Base52 文本还原成 SHA512-crypt 哈希；随后根据 README 中的自定义掩码用 Hashcat 找到压缩包密码；最后利用同一张图片在 iOS 与非 iOS 环境中的不同显示结果拼出二维码。比赛后期把这条链拆成了若干小题，但解法与原题一致。

## 解题过程

### Part 1：DNS 大小写信道与 Base52

流量中不断出现形如下面的 DNS 查询名：

```text
x.x.x.x.x.x.x.x.x.localhost
```

其中 `HGAME2022` 对应的请求是混淆项。有效查询的前 9 个字符其实是同一字母的大小写变化：前 8 位按大写为 `1`、小写为 `0` 转成一个 8 位整数，作为字符序号；第 9 位保留原字符，作为该序号上的数据。按序号排序后连接第 9 位即可得到只由 `A-Z` 和 `a-z` 组成的字符串。

下面的脚本复现了提取过程。官方 WP 以长度为 114 的包作为过滤条件；不同版本的 PyShark 字段名称可能略有差异，应以实际 PCAP 中相同那组 DNS 查询包为准：

```python
from Crypto.Util.number import long_to_bytes
import pyshark

capture = pyshark.FileCapture("password.pcapng")
records = []

for packet in capture:
    try:
        if int(packet.length) != 114:
            continue
        compact = packet.dns.qry_name.replace(".", "")
    except (AttributeError, ValueError):
        continue

    # HGAME2022 混淆项的前两个字符不同；有效项是同一字母的大小写组合。
    if len(compact) < 9 or compact[0].lower() != compact[1].lower():
        continue

    order_bits = "".join("1" if char.isupper() else "0" for char in compact[:8])
    records.append((int(order_bits, 2), compact[8]))

secret_text = "".join(char for _, char in sorted(records))

alphabet = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
value = 0
for char in secret_text:
    value = value * len(alphabet) + alphabet.index(char)

print(long_to_bytes(value).decode("ascii"))
```

Base52 解码结果是 Unix `crypt` 格式的口令哈希：

```text
$6$6ldh0bSoluytiIS2$vs6caYlhCSX1rmWaPrdDYI55hIg3Ls75PnbJmEBfYOsZRaknhS4cTdmyIbHPWz/2dTAdQUEM7IEmOc.GPo5UO.
```

### Part 2：SHA512-crypt 自定义掩码

哈希以 `$6$` 开头，对应 SHA512-crypt；Hashcat 使用模式 `1800`。README 的口令提示中出现了 `?1`、`?2`，而所谓“密钥”实际是两个自定义字符集，因此应使用掩码攻击：

```bash
hashcat -a 3 -m 1800 --force \
  '$6$6ldh0bSoluytiIS2$vs6caYlhCSX1rmWaPrdDYI55hIg3Ls75PnbJmEBfYOsZRaknhS4cTdmyIbHPWz/2dTAdQUEM7IEmOc.GPo5UO.' \
  --custom-charset1 'abcdefgnopqrs' \
  --custom-charset2 'hijklmtuvwxyz' \
  'co?2b?2nd_7?up?dm6Q_?1nd_4br?d9Pc'
```

破解结果为：

```text
combind_7Jp8m6Q_and_4br39Pc
```

去掉固定文字 `combind_` 与 `_and_`，把两段连接起来，得到压缩包密码：

```text
7Jp8m6Q4br39Pc
```

### Part 3：iOS 与非 iOS 双视图二维码

解压后得到一张带异常色块的图片。同一文件在普通图片查看器中只显示半张二维码，而经 iOS 图片显示/压缩链处理后会显出另一半。下面是官方 WP 保留的 iOS 显示结果；右上区域的二维码只是其中一半：

![iOS 系统显示出的二维码半图，需要与非 iOS 版本拼接](./HGAME2022-TOP-SECRET-wp/ios-half-qrcode.jpg)

实际操作时分别保存非 iOS 查看器与 iOS 设备中的显示结果，裁出两块二维码，按定位角和模块边界对齐拼接。扫描完整二维码即可获得最终 flag。官方 PDF 没有记录二维码解码文本，因此不能仅凭文档伪造最终字符串；但从 PCAP 提取、口令破解到双视图拼接的完整恢复路径均可由原附件复现。

## 方法总结

这道题把三种不同载体串成一条链：DNS 名称大小写承担顺序与数据，Base52 把哈希伪装成纯字母文本，Hashcat 掩码恢复下一阶段口令，最后再利用平台相关的图像显示差异隐藏二维码。处理长链题时，应在每一阶段记录“输入、变换、输出”并保存中间结果；这样即使后续附件缺失，也能准确指出证据断在何处，而不会把猜测当作最终 flag。
