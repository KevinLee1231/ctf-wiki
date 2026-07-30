# NepCTF2026 如烟大帝独断万古 Writeup

## 题目简述

附件是一篇表面正常的长篇小说，肉眼阅读没有明显线索。对 Unicode 字符类别统计后，可以发现正文中混入了 120 个 `U+FE00` 至 `U+FE0F` 的 Variation Selector。这 16 个连续码点恰好可以编码一个十六进制半字节。

本题的决定性障碍是识别不可见字符隐蔽信道并重组二进制帧，因此归入 Stego。

## 解题过程

先统计非间距标记和格式字符：

```python
from collections import Counter
from pathlib import Path
import unicodedata as ud

text = Path("novel.txt").read_text(encoding="utf-8")
marks = Counter(
    (f"U+{ord(ch):04X}", ud.name(ch, "?"), ud.category(ch))
    for ch in text
    if ud.category(ch) in {"Mn", "Cf"}
)
for entry, count in sorted(marks.items()):
    print(count, *entry)
```

结果中出现 120 个 `VARIATION SELECTOR-1` 至 `-16`。映射关系为：

```text
U+FE00 -> 0x0
U+FE01 -> 0x1
...
U+FE0F -> 0xF
```

按文本出现顺序提取，相邻两个 selector 分别作为高、低半字节：

```python
nibbles = [
    ord(ch) - 0xFE00
    for ch in text
    if 0xFE00 <= ord(ch) <= 0xFE0F
]
assert len(nibbles) % 2 == 0

frame = bytes(
    (nibbles[i] << 4) | nibbles[i + 1]
    for i in range(0, len(nibbles), 2)
)
```

120 个 selector 还原为 60 字节，头部是 `4e 45 50 01 33`。帧格式为：

```text
+----------------+------------+-------------+------------------+
| Magic (4 byte) | Length (1) | Payload (N) | CRC32 (4, BE)    |
+----------------+------------+-------------+------------------+
| "NEP\x01"      | 0x33 = 51  | UTF-8 flag  | payload 的校验值 |
+----------------+------------+-------------+------------------+
```

解析并校验：

```python
import zlib

assert frame[:4] == b"NEP\x01"
payload_len = frame[4]
payload = frame[5:5 + payload_len]
stored_crc = int.from_bytes(
    frame[5 + payload_len:9 + payload_len], "big"
)
assert len(frame) == 9 + payload_len
assert zlib.crc32(payload) == stored_crc == 0x6330A31E
print(payload.decode("utf-8"))
```

得到：

```text
NepCTF{var1at10n_s3l3ct0rs_h4unt_th3_n0v3l-afk6324}
```

## 方法总结

对“看起来只有普通文本”的隐写附件，应同时检查码点、Unicode 类别、双向控制符和不可见组合标记，而不是只搜索可打印字符串。题目还提供 magic、长度和 CRC32，说明还原后必须按二进制协议完整验证；这能排除 selector 顺序、半字节高低位或尾部截断错误。
